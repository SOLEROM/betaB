---
type: source
title: "Betaflight Docs — DisplayPort MSP Extensions"
source_type: documentation
author: Betaflight project
date_published: 2023+
url: https://betaflight.com/docs/development/API/DisplayPort
confidence: high
created: 2026-05-14
updated: 2026-05-14
tags: [source, msp, displayport, osd, digital-vtx]
key_claims:
  - "MSP_DISPLAYPORT is command ID 182 (0xB6)"
  - "Sub-commands: HEARTBEAT(0), RELEASE(1), CLEAR_SCREEN(2), WRITE_STRING(3), DRAW_SCREEN(4), OPTIONS(5), SYS(6)"
  - "WRITE_STRING payload: row(u8) + column(u8) + attribute(u8) + NULL-terminated string up to 30 chars"
  - "Attribute byte: bit 6 = blink, bits 0-1 = font number (4 fonts × 256 chars each)"
---

# Betaflight Docs — DisplayPort MSP Extensions

## Summary

Official Betaflight documentation page describing how the FC pushes OSD
characters to a digital VTX or goggles using MSP. This is the digital-video
side of the OSD architecture (the analog-video side uses [[MAX7456]] on the
FC itself).

## What it contributes

- The numeric command ID for `MSP_DISPLAYPORT` (182) and every sub-command
  with its payload layout.
- Confirms text is **NULL-terminated ASCII** placed by row/column —
  not a frame buffer.
- The attribute byte encoding for blink + font selection.
- The `MSP_DP_SYS` sub-command for positioning system-rendered elements
  (voltage, bitrate, warnings) — these are rendered by the goggles, not by
  the FC.

## Confidence

**High.** Primary source; matches the implementation in
`src/main/io/displayport_msp.c` in the betaflight repo.

## Related

- [[MSP DisplayPort]]
- [[OSD (On-Screen Display)]]
- [[MSP Protocol]]
