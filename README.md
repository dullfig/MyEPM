# cisco7940

A FreePBX 17 module to auto-provision Cisco 7940/7960 SIP phones — the "plug it in
and it just works" experience these phones never properly had, rebuilt for FreePBX 17
(Debian 12 / Asterisk 21, post-`chan_sip`).

> **Status: research / design phase.** No code yet. The full architecture lives in
> **[research/findings.md](research/findings.md)** — start there.

## The idea

Treat phone support like a **driver model** (detect → mint identity → provision →
assign → maintain), with the 7940 as the first driver:

- **Detect** a new phone the moment it appears — DHCP option 60/55 + MAC/OUI, then
  TFTP filename fingerprints.
- **Mint a per-MAC identity** at TFTP time, so a phone is *never anonymous* — the MAC
  is one unbroken thread of identity from DHCP → TFTP → SIP registration.
- **Park** unknown phones in a walled-garden context (registered but unassigned).
- **Climb the firmware ladder** — `OS79XX.TXT` as a moving pointer walks any ancient
  load up to current SIP, one rung per reboot.
- **Assign a user** from a GUI *or* by self-service: dial a config number from the
  phone, authenticate, and the phone provisions itself (Extension-Mobility style).
- **Realtime-backed** so the whole phone lifecycle is live DB writes — no Asterisk
  reloads, with a static dialplan reading per-phone settings live.

## Network shape

Physically segmented (CMMC): dumb switches, pfSense routing, a tri-homed PBX with the
phones on an isolated voice island (`ip_forward=0`). Isolation *is* the security
boundary — see findings §13.

## Where to start

`research/findings.md` §9 holds the open-items backlog / build order. The one
time-sensitive task is preserving the irreplaceable 7940 firmware images from the old
FreePBX 14 box (§7).
