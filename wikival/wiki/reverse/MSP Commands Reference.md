---
noteId: "d2076af04f9c11f194a2c3b1eecd91b7"
type: protocol
title: "MSP Commands Reference"
status: stub
direction: bidirectional
transport: serial | USB | CRSF
created: 2026-05-14
updated: 2026-05-14
tags: [protocol, reverse, msp, reference, stub]
---

# MSP Commands Reference

## Overview
<!-- Full table of MSP v1 + v2 command IDs, payload layouts, direction, and BF version availability. Currently partial in [[MSP Protocol]]; this page is the exhaustive ID lookup. -->

> [!gap]
> Stub. Pending: tabular dump of every ID from `msp_protocol.h`, `msp_protocol_v2_betaflight.h`, `msp_protocol_v2_common.h`, with payload schema and version-gated availability via [[MSP API Versioning]].

## Related
- [[MSP Protocol]] — protocol overview + frame format
- [[MSP v2 Frame Format]]
- [[MSP API Versioning]]
- [[MSP DisplayPort]]
- [[MSP over CRSF]]

## Source anchors
- `src/main/msp/msp_protocol.h` (V1 IDs)
- `src/main/msp/msp_protocol_v2_betaflight.h` (V2 BF IDs)
- `src/main/msp/msp_protocol_v2_common.h` (V2 common IDs)
- `src/main/msp/msp.c` (dispatcher)

## Sources
- [[betaflight-msp-protocol-h]]
- [[betaflight-deepwiki-msp]]
- [[inav-wiki-msp-v2]]
