# Cisco 7940 → FreePBX 17 Support Module — Research Findings

> Status: **research phase**. This is a design/notes document, not code.
> Goal: a FreePBX 17 module that lets a Cisco 7940 (and, later, other phones) be
> plugged into the network and auto-provisioned — "talk to phones" as the one job
> a PBX must do well. First driver = Cisco 7940 SIP.

---

## 1. Project goal

Build a FreePBX 17 module that:

1. Detects a new phone the moment it appears on the network.
2. Auto-provisions an unknown phone into a **parking context** (registered but
   unassigned, diagnosable).
3. Lets an admin **assign a user/extension** from a GUI, which regenerates the
   phone's config and pushes it live.

Framing that keeps the design clean: this is a **phone "driver" model**, like a
Windows PnP driver. Two halves every driver needs:

- **Bind/discovery** — steer a phone to *its* provisioner (DHCP option 150 + a
  dedicated segment). This is "device enumeration."
- **Model-specific provisioner** — the logic + templates that know a given
  phone's file formats and lifecycle. This is the "driver" proper.

The 7940 is the first driver. Adding Polycom/Yealink/etc. later = "register
another fingerprint + template against the same interface," not a rebuild.

Driver interface sketch: `detect(mac) → provision(state) → assign(user) → maintain(check-sync)`.

---

## 2. Network topology

Context / constraints (CMMC-driven, non-obvious):

- CMMC required network segmentation.
- Two Cisco **managed** switches were bought but turned out **EOL** (no viable
  maintenance contract — eye-watering cost). Abandoned.
- Segmentation is therefore **physical**: 6 Linksys **dumb** switches, one
  segment each. *This is actually stronger L2 isolation than VLANs* — separate
  switches are air-gapped at layer 2; nothing to misconfigure.
- **pfSense** routes between segments ("hairpinning") and faces the outside world.
- Cabling terminates on a **patch panel / plug board** (wired ~10 yrs ago,
  telephone-switchboard style) — re-segmenting a device = moving a patch cord.

### PBX box = the hub (3 legs)

```
                      ┌─────────────┐
   outside/trunk ─────┤             │  Leg 1 → pfSense (SIP trunk, internet; firewalled)
                      │  FreePBX 17 │
   newer phones  ─────┤    (PBX)    │  Leg 2 → newer-phone switch  (built-in TFTP)
                      │             │
   cisco 7940s   ─────┤             │  Leg 3 → Cisco switch        (custom TFTP)
                      └─────────────┘
```

Key consequences of phones hanging off the PBX (not off pfSense):

- **The PBX is the DHCP server for the phone segments** (pfSense can't — DHCP is
  link-local and pfSense isn't on those segments). → run **dnsmasq** on the PBX.
- **`net.ipv4.ip_forward = 0`** on the PBX → it terminates traffic per-NIC but
  won't bridge segments. This makes the **phones an island** whose only reachable
  host is the PBX (no route to data segment, no internet). That *is* the CMMC
  isolation story — host-enforced, no firewall rules to maintain. Also the
  control that makes multi-homing the PBX defensible to an assessor.
- **Bind services per-leg** (defense + function): custom TFTP → Cisco-leg IP only;
  built-in TFTP → newer-phone-leg IP only; pjsip → phone legs, not the outside
  leg (except the trunk). Nothing voice-related listens on the outside interface.
- **Media stays local**: phone↔PBX RTP never leaves the dumb switch (no hairpin,
  low latency). Only trunk media touches pfSense.

> No CDP/voice-VLAN needed. The original "advertise VLAN 100" idea was solving the
> TFTP-coexistence problem; the real solution is **separate IPs/segments**, not a
> VLAN — and on dumb switches a self-tagged VLAN would be fragile anyway. Dropped.

---

## 3. TFTP coexistence (don't evict FreePBX's built-in TFTP)

Problem: two TFTP daemons can't both bind `0.0.0.0:69`.

Solution: **separate by IP/segment.**

- FreePBX built-in TFTP = `tftpd-hpa` serving `/tftpboot` (Endpoint Manager writes
  configs there). NOT actually a FreePBX service — just an OS daemon. Leave it
  bound to the newer-phone-leg IP, untouched.
- Custom Cisco TFTP daemon binds to the Cisco-leg IP only.
- Both happily on port 69 because different IPs.

### DHCP (dnsmasq) steering

- dnsmasq does **DHCP + DNS** in one small daemon; per-interface config.
- **Option 150 is NOT universal — it's the Cisco one.** Hand out the right option
  per leg (segments are physically separate, so no vendor-class gymnastics):
  - **Cisco leg** → `option 150` (Cisco-proprietary, *list of IPs*) = custom-TFTP IP
  - **newer-phone leg** → `option 66` (RFC-standard, *string*; can be host/IP/URL)
    = built-in-TFTP server. (Earlier draft wrongly said 150 here — generic phones
    use 66.) Add `option 160` (URL) on a Polycom segment.
  - Belt-and-suspenders if a segment is mixed: hand out 66 **and** 150 (+160); each
    phone reads the one it understands. Cisco checks 150 first, then 66.

| Phone family | Primary | Also | Format | Transport |
|---|---|---|---|---|
| Cisco (7940, 79x1) | **150** | 66 | IP list | TFTP |
| Generic/RFC | **66** | 67 | string | TFTP/HTTP |
| Polycom | **160** (URL) | 66, 43 | string/URL | HTTP/TFTP |
| Yealink | **66** (URL) | custom | string/URL | HTTP/TFTP |
| Grandstream | **66** | 43 | string/URL | HTTP/TFTP |
| Snom | **66** (URL) | 43 | string/URL | HTTP/TFTP |
| Aastra/Mitel | **66** | — | string | TFTP |

  > Option 66 is a *string* and can hold a full `http(s)://` URL — that's how the
  > HTTP-provisioning phones (Polycom/Yealink/Snom) ride it. dnsmasq:
  > `dhcp-option=150,<ip>` · `dhcp-option=66,"<str>"` · `dhcp-option=160,"<url>"`.
- Hygiene: `interface=` + `bind-interfaces` so dnsmasq listens only on phone legs,
  never the outside leg.
- **Leave dnsmasq's own TFTP OFF** (`enable-tftp` unset) — it only serves static
  files (can't do per-MAC dynamic generation) and would fight port 69.

Three cooperating OS-level pieces, FreePBX/Asterisk above them:
`dnsmasq` (DHCP/DNS) · `tftpd-hpa` (newer phones) · custom daemon (Cisco).

---

## 4. Detection / fingerprinting

Two sensors feeding one state DB:

### Sensor A — DHCP (earliest signal: fires before any TFTP)

dnsmasq `dhcp-script` hook runs on every lease add/old/del with MAC, IP, hostname,
vendor-class in env vars → **this is the "new kid in town" trigger**, native.
On a new lease from a phone OUI/VCI, **pre-create the parking record**.

Fingerprint fields in the DISCOVER/REQUEST:
- **Option 60 (Vendor Class Identifier)** — plaintext self-ID, often brand+model.
  7940 ≈ `Cisco Systems, Inc. IP Phone CP-7940G` *(exact string firmware-dependent
  — CAPTURE FROM PCAP)*. Polycom `Polycom-VVX…`, Yealink `yealink SIP-T46G`, etc.
- **Option 55 (Parameter Request List)** — ordered option set; the classic DHCP
  fingerprint (Fingerbank/Satori key on this). Phone asking for 150 + 132 ⇒ VoIP.
- **chaddr** — the MAC ⇒ OUI.

Caveat: option 60 is self-reported (spoofable, sometimes blank on old firmware).
Trusted segment, so fine — but corroborate with OUI. DHCP IDs the *hardware*, not
the *firmware mode* (7940 can be SCCP or SIP) — TFTP disambiguates that.

### Sensor B — TFTP filename signatures

Each vendor's bootloader blurts a hardcoded request sequence. Match on **presence
of a signature file**, NOT exact ordering (firmware versions reorder; security
files requested first then 404'd is normal).

| Vendor / family        | Signature request(s)                              | Tell |
|------------------------|---------------------------------------------------|------|
| Cisco 7940/7960 (SIP)  | `OS79XX.TXT`, `SIPDefault.cnf`, `SIP<MAC>.cnf`     | UPPERCASE MAC, `.cnf` |
| Cisco 79x1 / SCCP      | `CTLSEP<MAC>.tlv`, `ITLSEP<MAC>.tlv`, `SEP<MAC>.cnf.xml` | `SEP`/`CTL`, `.tlv` |
| Yealink                | `y000000000000.cfg`, `y0000000000XX.cfg`, `<mac>.cfg` | leading `y` + model code |
| Polycom                | `<mac>.cfg`, then `000000000000.cfg` on 404       | all-zeros master |
| Grandstream            | `cfg<mac>`, `cfg<mac>.xml`                         | `cfg` prefix, binary |
| Aastra/Mitel           | `aastra.cfg`, `<mac>.cfg`, `startup.cfg`          | `aastra.cfg` |
| Snom                   | `snom<model>.htm/.xml`, `<mac>.xml`               | `snom`+model |

### OUI (the strong, early signal)

MAC is embedded in most provisioning filenames AND in the DHCP lease, so the OUI
(first 3 octets, IEEE vendor block) is available immediately.

- OUI = **vendor ID**; filename shape = **device/dialect ID**. Use together.
- A vendor owns **many** OUIs (Polycom: `00:04:f2`, `64:16:7f`, `48:25:67`,
  `00:e0:db`…; Cisco: dozens). → ship a small static OUI→vendor table; curate only
  supported brands; refresh ~yearly from IEEE registry.
- OUI gives **brand, not model** — refine model from filename (Yealink encodes
  model in `y0000000000XX.cfg`) or later requests.
- **Cross-check, don't just decide**: Polycom OUI asking for a Yealink-shaped file
  = alarm, log+flag, don't provision.
- **Locally-administered bit** (2nd-LSB of first octet) set ⇒ OUI meaningless;
  one-line sanity guard against spoofed/virtual MAC.

> Disambiguation note: Aastra and Polycom both request `<mac>.cfg`. DON'T wait for
> Polycom's `000000000000.cfg` to disambiguate — Polycom only asks for it when
> `<mac>.cfg` *fails*, so if you're serving `<mac>.cfg` you never see it. Use the
> **OUI** (already in the filename/lease) to decide on the first request.

Decision rule: *resolve MAC (filename else lease) → OUI lookup → vendor; filename
shape → dialect/model; corroborate; DB lookup → new vs known.*

---

## 5. Cisco 7940 cold-boot timeline (the first driver, fully specified)

Three decision points, each owned by one component:

```
1. Link up
   └─ Phone asks for voice VLAN via CDP. Dumb switch never answers → phone
      RETRIES CDP for ~2 min, times out, THEN proceeds untagged on native.
      Fix: a controlled CDP responder on the phone-leg (see §5.1) kills the wait.

2. DHCP DISCOVER → REQUEST → ACK
   └─ dnsmasq hands IP + option 150 = custom-TFTP IP
   └─ dhcp-script fires  ── DECISION 1: ENROLL ── state = new/parked   ◄ "new kid"

3. TFTP (to the custom daemon), roughly:
   ├─ CTLSEP<MAC>.tlv     security file; virgin phone 404 is harmless
   ├─ OS79XX.TXT          "which firmware load?" ── DECISION 2: FIRMWARE LADDER
   │                       serve NEXT rung; phone flashes→reboots→re-DHCP→loops
   │                       repeat until phone is on the FINAL SIP load
   ├─ P0S3-…              firmware image files; static, only when flashing
   ├─ SIPDefault.cnf      global defaults (proxy=FreePBX, codecs, NTP); generated
   ├─ SIP<MAC>.cnf        ── DECISION 3: PARK vs REAL ──
   │                       new/parked → parking template (label UNPROVISIONED <MAC>)
   │                       assigned   → real template (user ext/creds/display)
   └─ dialplan.xml        generated

4. SIP REGISTER → FreePBX (chan_pjsip) → lands in parking OR as real extension

5. Admin assigns user in GUI → DB flips to `assigned`
   └─ push SIP NOTIFY  Event: check-sync  → phone reboots, re-reads SIP<MAC>.cnf
      → comes back as the real extension. (7940 SIP firmware honors check-sync.)
```

Symmetry: **newness decided once (DHCP); firmware walked at `OS79XX.TXT`; config
decided once (`SIP<MAC>.cnf`); park→real = one DB flip + one check-sync.**

7940 is mercifully flat: `SIP<MAC>.cnf` IS the settings (no Polycom-style
master/pointer indirection). Easy first driver.

### 5.1 The CDP / voice-VLAN boot wait (and a loose end)

7940/7960 are **CDP-era** (predate LLDP-MED). On boot the phone asks for a voice
VLAN via **CDP**. Dumb switch never replies → phone retries for **~2 min**, times
out, then proceeds untagged on native. Pure UX pain, not a functional blocker —
but it kills the "plug and play" feel on every cold boot.

**Loose end (history):** Dan previously ran a **listener handing out (bogus) VLAN
assignments** — almost certainly a CDP responder to short-circuit this wait. It
**may still be running on the old FreePBX 14 box** and may be *load-bearing* (part
of why phones work today). Also a **rogue-L2 / CMMC audit smell** (unmanaged daemon
advertising topology). ACTION: find it BEFORE decommissioning the old box —
`ss -lnp`, `ps`, cron, `/etc/rc.local`; rule out a stray daemon (`lldpd`/`cdpd`/
custom) and **DHCP option 132** (VLAN-ID, a different mechanism for the same goal).
Document, then kill or fold into the controlled design.

**Forward design — repurpose, don't reintroduce tagging:** The PBX's phone-leg IS
L2-adjacent on the phones' dumb switch, so it CAN emit CDP the phones hear (the
earlier "advertiser needs L2 adjacency" objection doesn't apply here). Goal is the
OPPOSITE of the old "advertise VLAN 100 + tag" idea — we do NOT want tagging on
dumb switches. Instead **answer CDP with no voice-VLAN / native** so the phone
concludes "nothing to wait for" and proceeds untagged → fast boot.

- Mechanism: **`lldpd`** (Debian-packaged) can emit **CDP** (not just LLDP), bound
  to the phone-leg interface only (same per-interface hygiene as the rest).
  Advertises no voice-VLAN policy. One more cooperating daemon — the *controlled,
  documented* version of the mystery listener above.
- Manual fallback for a bench phone today: **Settings → Network Configuration →
  disable CDP / clear Operational VLAN** so it won't wait. Per-phone; doesn't scale
  to plug-and-play, but unblocks one phone now. (Whether this is pushable via config
  vs. menu-only on the 7940 = VERIFY.)

---

## 6. Firmware ladder (the "re-brain to current SIP" mechanism)

Why an old phone can't jump straight to current SIP firmware:

- **`OS79XX.TXT` is a moving pointer.** One line = the app load the phone *should*
  run. Each boot the phone compares it to its *current* load: same → proceed;
  different → download/flash/reboot/restart. You never address the phone's current
  version directly — you keep changing the target and it chases.
- **Can't point straight at the final load**, because the old bootloader can't
  *accept* a modern image in one hop. Two gaps force intermediate rungs:
  1. **Format/signing gap** — newer loads are signed (`.sbn`) and an old loader
     can't parse/trust them until a transitional load teaches it the format.
  2. **SCCP→SIP conversion** — flipping skinny→SIP (`P003…` → `P0S3…`, S = SIP) is
     itself a bridged step.
- Path is a **ladder**: very-old skinny → bridge load(s) → SIP-capable → current
  SIP. Skip a rung → flash fails or loops.

Drops into the module as a small state machine — **the reboot loop we already have
IS the clock that advances it**:

- **Static ordered ladder table** — the known-good rungs (THIS is the old hack).
- **Per-MAC "current rung"** field in the same DB record dhcp-script created.
- On each `OS79XX.TXT` request: serve the next rung; when the phone climbs it
  (re-DHCP/re-TFTP), bump the field; repeat until final SIP, then DECISION 3 applies.

---

## 7. ⚠ Firmware preservation (most time-sensitive item)

The *mechanism* is reconstructable; the firmware **images are not**. 7940 is long
EOL; loads were login-walled behind Cisco contracts that may no longer exist; the
intermediate bridge versions are exactly what nobody mirrors.

- **SOURCE IDENTIFIED: the old FreePBX 14 box is still live** and running the
  phones — it holds the irreplaceable `/tftpboot`. (New box = Dell R640 w/ FreePBX
  17; CMMC forced the hardware refresh, which luckily kept the old box intact.)
- **Copy method (do NOT drag onto a Windows USB — case-mangling risk):** Cisco
  filenames are case-SIGNIFICANT (`OS79XX.TXT`, `SIP<MAC>.cnf` are UPPERCASE);
  FAT/exFAT/Windows is case-insensitive and can flatten them. `tar` first to
  preserve names/case/perms (reading is safe on the live box):
  ```
  tar czf /tmp/tftpboot-backup.tgz /tftpboot      # firmware ladder + known-good cnf
  tar czf /tmp/asterisk-backup.tgz /etc/asterisk  # chan_sip "Rosetta stone" (see below)
  ```
  Then copy the archives to **two** places (USB + cloud/second drive).
- **Also grab `/etc/asterisk`** (esp. `sip.conf` / `sip_additional.conf`): FreePBX
  14 = chan_sip era; that config is the answer key for the chan_pjsip translation
  (exact secrets, codecs, DTMF, NAT that worked with these phones).
- `/tftpboot` is a twofer: the irreplaceable **firmware ladder** AND a **known-good
  `SIP<MAC>.cnf` / `SIPDefault.cnf` / `OS79XX.TXT`** = the basis for module templates.
- Lead to chase: someone posted these on **voip-info.org** (or similar VoIP forum)
  years ago — check for a surviving mirror as a backup-to-the-backup.
- When the files are found, **document the exact rung list** (versions, order,
  where the SCCP→SIP flip happens). Dan may already have documented it ~15 yrs ago.

---

## 8. Asterisk / FreePBX 17 notes

- FreePBX 17 = Debian 12, PHP 8.2; ships Asterisk 20/21. **`chan_sip` was removed
  in Asterisk 21** — likely a big part of what "hosed" the old 7940 setup (if it
  registered via chan_sip). 7940 must now use **`chan_pjsip`**. *VERIFY which
  Asterisk version this FreePBX 17 installed.*
- pjsip settings to verify for 7940: digest/MD5 auth, **no SRTP**, single AOR/
  contact, `allow=ulaw,alaw`, DTMF rfc2833, no ICE, direct_media off, and list
  each phone subnet as **`local_net`** so internal traffic isn't NAT-mangled
  (NAT/external settings scoped to the trunk side only).
- Module anatomy (FreePBX 17 / BMO): `module.xml`, install/uninstall hooks, a GUI
  page, and a way to run/own the custom daemon + dnsmasq stanza. *Research the
  current BMO module skeleton for v17.*

---

## 9. Open items (verify against real hardware / decide)

- [ ] **Locate & preserve the 7940 firmware files** (§7) — P0, time-sensitive.
- [ ] **Find the old CDP/VLAN listener** on the FreePBX 14 box before decommission
      (§5.1) — may be load-bearing; rule out DHCP option 132 too.
- [ ] Decide/stand up a controlled **CDP responder (`lldpd`)** on the phone-leg to
      kill the ~2-min voice-VLAN boot wait (§5.1).
- [ ] Capture the exact **firmware ladder** (rung versions + order + SCCP→SIP point).
- [ ] PCAP a cold 7940 boot: exact **option-60 string**, **option-55 PRL**, and the
      precise **TFTP file request order** for its firmware revision.
- [ ] Confirm which **Asterisk version** FreePBX 17 installed (chan_pjsip path).
- [ ] Nail down **chan_pjsip endpoint config** that registers + passes audio on 7940.
- [ ] Decide dispatcher scope: **actively serve unknown vendors** (general driver
      bus day one) vs **detect+log non-Cisco, fully drive only 7940** for now.
- [ ] Confirm who runs DHCP today / migrate to **dnsmasq on the PBX**; per-leg
      option-150 values + `bind-interfaces`.
- [ ] Curate the **OUI→vendor** table for supported brands.
- [ ] FreePBX 17 **BMO module skeleton** — minimal working module + GUI page.
- [ ] Decide the **parking context** design (extension range, labeling, how it
      surfaces in the GUI as "unassigned phones"). Park each phone as a UNIQUE
      per-MAC endpoint (enables IVR self-onboarding, §12) + walled-garden dialplan.
- [ ] **IVR self-provisioning** (§12): decide secret model — admin one-time-code vs
      user-self-onboard (own PIN) vs both. Then build dialplan + AGI + audit log.
- [ ] **Realtime backing** (§14): verify FreePBX 17 leaves `sorcery.conf` /
      `extconfig.conf` unmanaged; confirm Sorcery multi-wizard syntax for stacked
      flat+realtime endpoints; prototype a `park-<MAC>` realtime endpoint coexisting
      with FreePBX endpoints (no reload).
- [ ] Decide the **pre-created FreePBX extension pool** approach for assigned phones
      (pool size, naming, how the GUI maps a claimed phone → an existing extension).

---

## 10. Component inventory (target box)

| Component      | Role                                              | Notes |
|----------------|---------------------------------------------------|-------|
| dnsmasq        | DHCP + DNS for phone legs; dhcp-script "new kid"  | TFTP OFF; bind to phone legs |
| tftpd-hpa      | built-in TFTP for newer phones (`/tftpboot`)      | bound to newer-phone-leg IP |
| custom daemon  | dynamic TFTP for Cisco: fingerprint + ladder + park/real | bound to Cisco-leg IP |
| Asterisk 20/21 | call handling; chan_pjsip endpoints               | local_net per phone subnet |
| FreePBX module | GUI: assign user → flip state → check-sync; owns templates + state DB | BMO |
| lldpd          | CDP responder on phone-leg → kills ~2-min voice-VLAN boot wait | emits CDP; no voice-VLAN/native; bound to phone leg |
| pfSense        | inter-segment routing, trunk/outside, firewall    | not on phone segments |

---

## 11. GUI / module UX

**Architectural spine:** everything in the GUI is a window onto the **one state DB**
the daemons already maintain. dhcp-script *writes* new phones in; TFTP dispatcher
*renders* from it; GUI *reads/edits* it; "assign user" = a write + a check-sync push.
One table at the center, three actors around it → the GUI does no discovery of its
own, it's just a view. (Clean MVC: daemons write state, dispatcher renders state,
GUI edits state.)

Screens:

1. **Detected Phones** (the heart) — live table from the state DB:
   `MAC | Vendor(OUI) | Model | IP | State | Firmware rung | Assigned to | first/last seen`
   - State lifecycle: `new → parked → firmware-upgrading → assigned`.
   - Row actions: **Assign user** (FreePBX extension dropdown), **Re-park**,
     **Reboot** (push `check-sync`), **Forget**, **View request log** (exact TFTP
     files requested — key for debugging a phone that won't provision).
   - New phones appear automatically the instant they DHCP.
2. **Settings (per phone-segment)** — interface, **IP range/subnet** (dnsmasq
   dhcp-range), gateway (or none = island), lease time, **which provisioning
   options to hand out** (150/66/160, defaulted per §3 table), global SIP defaults
   (proxy=FreePBX, codecs, NTP, timezone).
3. **Firmware ladder** — upload rescued firmware files, set **rung order**, mark the
   **final SIP load**. (Where tomorrow's `/tftpboot` haul lands; makes the ladder a
   managed/visible thing, not tribal knowledge.)
4. **Templates** — editable per-driver provisioning templates with token
   substitution (`{EXTENSION}`,`{SECRET}`,`{DISPLAYNAME}`,`{PROXY}`), seeded from the
   rescued known-good config.
5. **Service health / diagnostics** — up/down for dnsmasq, tftpd, custom daemon,
   lldpd; recent TFTP requests; DHCP leases.
6. *(future)* **Driver registry** — installed drivers + fingerprints, enable/disable
   (the general "driver bus" view).

---

## 12. Self-service provisioning from the phone (IVR onboarding)

**Idea:** a parked/unprovisioned phone can dial a **config number**, authenticate,
and **self-assign its extension** — no admin at the GUI. Zero-touch onboarding;
essentially Cisco "Extension Mobility" reimplemented on the 7940.

**Architectural fit:** the IVR is just a **4th actor writing the same `assign`
transition** to the state DB (GUI is the others' front door; IVR is another). Same
regen + check-sync path → almost no new structure.

**Mechanics:**
1. **Park each phone as a UNIQUE per-MAC pjsip endpoint** (`park-<MAC>` + generated
   per-MAC secret). → on any call, the endpoint name maps back to the MAC, so the
   IVR knows exactly which physical phone is self-assigning. (Refines parking: per-MAC
   creds, NOT a shared generic login.)
2. **Parking context = walled garden:** the ONLY dialable destination is the config
   number (reserved feature code). No outbound, no other extensions. UX + containment.
3. Dial config number → Asterisk IVR collects DTMF (`Read()`) → **AGI** script
   (PHP/Python) bridges dialplan ↔ module DB.
4. AGI validates secret → flips record to `assigned` (ext X) → regen config → fire
   `check-sync` NOTIFY → phone reboots as real extension. IVR reads back confirmation.

**Security (CMMC — this is an auth/access-control event):**
- ❌ **Single global provisioning PIN** — shared static secret, no individual
  accountability. Lab only, not the CMMC network.
- ✅ **Admin pre-stages + one-time code** — admin marks ext "claimable" in GUI, system
  generates a one-time code, tech claims with it. Separation of duties, no reuse,
  logged. Best for new phones / new hires.
- ✅ **User self-onboard w/ own credential** — user enters their extension + personal
  PIN (voicemail/directory). Authenticates as self; per-user audit. Extension-Mobility
  style; best for re-homing to a new desk.
- The two ✅ models can coexist. Always add: **confirmation readback**, **lockout after
  N bad tries** (DTMF PIN brute-force defense), **full audit log** (MAC, ext, secret,
  timestamp — free, since we're already writing the DB).

**OPEN DECISION:** admin-one-time-code vs user-self-onboard vs both → changes GUI
pre-staging + IVR prompts.

**Implementation notes:** dialplan context `[cisco-parking]`; IVR via `Read()`+AGI
(or ARI/Stasis for richer flow); check-sync via `pjsip send-notify` / AMI; record or
TTS the prompts.

---

## 13. The identity chain (conceptual keystone)

The whole module rests on one idea: **there are no anonymous phones.** Every phone
gets a MAC-derived identity *before it ever sends a SIP packet*, because we control
the config it boots with. "Parked" is not pre-registration — it IS a registration,
just under a **provisional per-MAC identity we minted**. The TFTP dispatcher is the
moment identity is minted.

So the old hard question — *"how does Asterisk know which phone is calling before
it's provisioned?"* — dissolves: we TOLD the phone who to be. The **MAC is one
unbroken thread of identity**, carried by a different mechanism at each hand-off but
never dropped:

```
DHCP      chaddr (the MAC itself)        → enroll
TFTP      filename / lease → MAC         → mint per-MAC SIP creds into the config
Register  phone comes up as park-<MAC>   → SIP identity traceable to the MAC
IVR call  endpoint name = park-<MAC>     → Asterisk recovers the MAC
Assign    flip to real extension          → re-mint identity, check-sync
```

**Security corollary (CMMC win):** because every phone authenticates under a minted
per-MAC credential, **anonymous/guest SIP can be disabled entirely** — no chan_sip
`allowguest`, no pjsip anonymous endpoint, no identify-by-source-IP (flaky under
DHCP anyway). There is never an unidentified device to catch, so the insecure
catch-all is simply turned off. Tighter and simpler.

### 13.1 Threat model & security posture (write this for the assessor)

Defeats the **most common SIP attack**: remote registration / toll-fraud scanners
(sipvicious et al.). But be precise about WHY — the strength is layered, and the
identity is NOT the secret:

- **DO NOT rely on identity obscurity.** A MAC is public (ARP, on-wire, sticker on
  the phone), so `park-<MAC>` is guessable. Obscurity = speed bump, not a wall.
- **Layer 1 — network isolation (the strong wall).** Physical island: dumb switch,
  no inter-segment routing, `ip_forward=0`. Internet/LAN attacker has NO IP path to
  the SIP port — must physically plug into the voice switch to even try. Defeats
  ~all remote SIP attacks before identity matters.
- **Layer 2 — no anonymous SIP + per-endpoint secret.** Even a device on the segment
  can't register without valid creds. The wall is the **secret**, not the username.
- **Layer 3 — walled-garden parking + provisioning secret.** A registered-but-
  unassigned identity can only dial the IVR, which needs a one-time code / user PIN
  (§12) to claim an extension. Lateral movement contained.

**Two guardrails that must not be violated:**

1. **Per-endpoint secret MUST be random, stored in the DB — NOT derived from the MAC.**
   `secret = hash(MAC)` ≡ public secret (MAC is public) → whole scheme collapses.
2. **TFTP leaks the secret → isolation is the real crypto boundary, not polish.**
   `SIP<MAC>.cnf` is served cleartext + unauthenticated; anyone on the voice segment
   can fetch another phone's config and read its password. 7940 era has no fix (no
   HTTPS provisioning / usable config encryption). ∴ physical access control to the
   voice switch IS the cryptographic boundary — treat it as such.

**Residual threat:** an attacker with physical access to the voice segment. Mitigated
by Layer 3 + physical security. Document this honestly; don't claim the secret is
private against a local attacker — it isn't.

---

## 14. Backing store: Asterisk Realtime "behind FreePBX's back" (DECISION)

FreePBX compiles its MySQL config to **flat `.conf` files + `reload`** (the "press
the button for every change" model). That's defensible for **dialplan** (FreePBX's
huge generated dialplan; realtime dialplan is slow) but annoying for **endpoints**,
where Asterisk Realtime (ARA) shines — `INSERT` a row, phone registers live, no
reload. Goal: get realtime liveness for the Cisco provisioning layer WITHOUT
fighting FreePBX.

**Mechanism (legit):** pjsip loads config via **Sorcery**, which supports **stacked
backends** for the same object type → flat-file endpoints (FreePBX) AND realtime
endpoints (us) coexist in ONE Asterisk. Wire via `sorcery.conf` + `extconfig.conf` +
`res_odbc.conf`. **FreePBX does NOT regenerate `sorcery.conf`** (VERIFY on 17) → our
wiring isn't clobbered. Realtime endpoints are looked up on demand → **immune to
FreePBX reload churn.** (Verify exact `sorcery.conf` multi-wizard syntax.)

**Discipline — split by lifecycle state:**

- **Parked phones → realtime, fully behind FreePBX's back.** Dispatcher INSERTs a
  `park-<MAC>` endpoint/auth/aor into our tables at TFTP time → phone registers live,
  no reload, into the walled-garden parking context. FreePBX never sees these
  ephemeral devices (correct — they don't belong in its world). This is where
  realtime genuinely wins.
- **Assigned phones → through the FRONT DOOR (not behind the back).** Do NOT
  realtime-define a number FreePBX also owns (double definition → chaos; inbound
  breaks because FreePBX's dialplan only routes `Dial(PJSIP/<ext>)` to extensions IT
  created). Instead **pre-create a pool of normal FreePBX extensions**; "assign" =
  hand the phone an EXISTING extension's creds via TFTP config + check-sync. FreePBX
  fully owns the real extension (voicemail/BLF/features/dialplan); STILL no reload
  because the extension already existed.
- **Parking dialplan (context + config-IVR feature code) → `extensions_custom.conf`**
  — the blessed seam FreePBX `#include`s and NEVER overwrites. Not behind its back;
  the sanctioned extension point.

Net: realtime-behind-the-back where FreePBX is weak (ephemeral parking), FreePBX
front-door where it's strong (permanent extensions), `_custom.conf` for the glue.

**Landmines:**
- **Namespacing non-negotiable** — parked endpoints use names FreePBX would never
  mint (`park-<MAC>`); never realtime-define a FreePBX-owned number.
- **Two sources of truth** — realtime endpoints don't show in FreePBX GUI (or in
  `pjsip show endpoints` until referenced). Our GUI is their home; discipline cost.
- **Maintenance tax** — FreePBX major upgrades may shift `sorcery.conf`/`extconfig.conf`
  assumptions. Pin it; test upgrades in a snapshot first.

**Convergence:** the "state DB is the spine" (§11) and realtime collapse into ONE
object — if the parking layer of the state DB **is** the pjsip realtime tables, then
"mint identity at TFTP time" (§13) is a single write that both generates the phone's
config AND makes Asterisk ready to accept its registration. Spine == Asterisk config.

### 14.1 Static dialplan + live data reads (dissolves the entanglement objection)

The dialplan can read realtime/DB data **at call time**, so a generic compiled
dialplan can drive per-phone features WITHOUT regeneration:
- **`func_odbc`** (workhorse): `${ODBC_FN(${MAC})}` ← arbitrary SQL.
- **`REALTIME(family,col,val,field)`**: live read of a realtime family.
- **`${DB(family/key)}`**: Asterisk's live AstDB key-value store.

**FreePBX already uses this pattern** — call-forward / DND / follow-me are live state
in AstDB (`${DB(CF/${EXTEN})}` etc.), read by the *compiled* dialplan; that's why
`*72` works with no Apply Config reload. FreePBX's reload is really only for
*structural* change (an extension's existence/endpoint/mailbox), NOT mutable
settings. So the "endpoint is entangled with generated dialplan" objection (§14) is
narrower than it looks — and it **dissolves for our module because we own our
dialplan**: write one generic Cisco dialplan in `extensions_custom.conf` (loaded
once), reading per-phone behavior (assigned extension, forwarding, VM box, park-vs-
assigned) live from realtime. Reassign/forward/re-park = pure DB writes, **zero
reload ever**. Liveness extends from registration → features. Spine drives routing too.

**Caveat:** routing/CF via `func_odbc`/`DB()` is rock-solid. **Voicemail-by-realtime**
(`voicemail` family + ODBC storage + MWI) is the finicky family → PROTOTYPE & TEST,
don't assume.

**Refines the §14 assigned-phone fork:** assigned phones can EITHER roll their own
live dialplan reading realtime (max independence, but reimplements CF/VM/etc.) OR use
the pre-created FreePBX pool (full feature stack + GUI for free, within FreePBX's
model). Reuse likely wins for real extensions (don't reinvent voicemail); DIY door is
open. Parked phones: generic-dialplan-reads-realtime is perfect (≈no features needed).
