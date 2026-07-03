# MyEPM

A homegrown FreePBX 17 endpoint-manager module for **Polycom VVX 450** phones —
built after concluding that the commercial EPM was not worth the aggravation.

> **Status: research / design phase.** No code yet.

## History

This repo began life as `cisco7940`, a provisioning module for ancient Cisco
7940/7960 SIP phones. The phones were replaced with Polycom VVX 450s, FreePBX was
upgraded to 17 (Debian 12 / Asterisk 21), and the commercial EPM was tried and
rejected — so the project pivoted to a Polycom module.

The original architecture research lives in
**[research/findings.md](research/findings.md)**. Its core "driver model" is
phone-agnostic and carries over:

- **Detect** a new phone the moment it appears (DHCP fingerprints + MAC/OUI).
- **Mint a per-MAC identity** at provisioning time — the phone is never anonymous.
- **Park** unknown phones in a walled-garden context (registered but unassigned).
- **Assign a user** from a GUI or by self-service from the phone itself.
- **Realtime-backed** — the phone lifecycle is live DB writes, no Asterisk reloads.

What changes for the VVX 450: provisioning moves from TFTP + `OS79XX.TXT` firmware
ladders to Polycom's UC Software model — an HTTP(S) provisioning server serving
`000000000000.cfg` / per-MAC `<MAC>.cfg` XML files, with firmware updates handled
the same way.

## Network shape

Physically segmented (CMMC): dumb switches, pfSense routing, a tri-homed PBX with
the phones on an isolated voice island (`ip_forward=0`). Isolation *is* the
security boundary — see findings §13.
