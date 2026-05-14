---
noteId: "d20743e14f9c11f194a2c3b1eecd91b7"
type: protocol
title: "CRSF Protocol"
status: stub
direction: bidirectional
transport: serial (UART, 420kbps typical)
created: 2026-05-14
updated: 2026-05-14
tags: [protocol, reverse, crsf, expresslrs, tbs, stub]
---

# CRSF Protocol

## Overview
<!-- Crossfire Serial Protocol — TBS/ELRS bidirectional radio link to FC. Carries RC channels, telemetry, MSP tunneling, link statistics. -->

> [!gap]
> Stub. Pending: frame format (`sync`, `len`, `type`, `payload`, `crc8`), full frame-type table, link-stats payload. MSP-over-CRSF chunking already documented in [[MSP over CRSF]].

## Related
- [[MSP over CRSF]] — MSP_REQ/RESP/WRITE tunnel framing
- [[ExpressLRS]]
- [[CRSF_BAUDRATE]]

## Source anchors
- `src/main/rx/crsf.c`, `src/main/rx/crsf_protocol.h`
- `src/main/telemetry/crsf.c`

## Sources
- [[betaflight-crsf-protocol-h]]
