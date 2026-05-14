---
noteId: "d207b9114f9c11f194a2c3b1eecd91b7"
type: protocol
title: "OSD Font Format"
status: stub
direction: host-to-fc
transport: SPI (MAX7456) | MSP DisplayPort (digital VTX)
created: 2026-05-14
updated: 2026-05-14
tags: [protocol, reverse, osd, font, max7456, stub]
---

# OSD Font Format

## Overview
<!-- MAX7456 character ROM format (54 bytes per glyph, 256-glyph NVM) and how Betaflight Configurator writes custom fonts. For digital VTX the font is rendered goggle-side; BF only sends glyph indexes via [[MSP DisplayPort]]. -->

> [!gap]
> Stub. Pending: MAX7456 NVM byte layout, character map, BF font-upload MSP commands, digital VTX font-number bits in DisplayPort attribute byte (bits 0-1).

## Related
- [[MAX7456]]
- [[OSD (On-Screen Display)]]
- [[MSP DisplayPort]]

## Source anchors
- `src/main/drivers/max7456.c`, `src/main/osd/osd.c`

## Sources
-
