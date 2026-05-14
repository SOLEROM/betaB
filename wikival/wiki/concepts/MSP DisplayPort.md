---
type: concept
title: "MSP DisplayPort"
status: documented
created: 2026-05-14
updated: 2026-05-14
tags: [concept, msp, osd, displayport, digital-vtx]
related:
  - "[[OSD (On-Screen Display)]]"
  - "[[MSP Protocol]]"
  - "[[MSP v2 Frame Format]]"
---

# MSP DisplayPort

## Overview

MSP DisplayPort is the mechanism Betaflight uses to push OSD characters to a
**digital video transmitter** (DJI O3 Air Unit, Walksnail Avatar, HDZero) or
to a goggle that renders OSD itself. It replaces the analog-video path that
went through the on-FC [[MAX7456]] chip with a serial protocol over UART.

The FC sends ASCII strings positioned by row/column; the goggles (or VTX)
do the actual rendering with their own font. The result is a clean DVR, no
character-set baked into video, and richer fonts than the MAX7456's 256-char
EEPROM.

(Source: [[betaflight-displayport-api]])

## Command ID

- `MSP_DISPLAYPORT = 182` (`0xB6`) — V1 command, single-byte ID.

The first payload byte is the **sub-command**. The remaining bytes depend
on the sub-command.

## Sub-Commands

| Sub-cmd | ID | Purpose | Payload after sub-cmd byte |
|---------|----|---------|----------------------------|
| `HEARTBEAT` | 0 | Keepalive — prevents "disconnected" message on slave OSD boards | none |
| `RELEASE` | 1 | Clear display and return device to local rendering | none |
| `CLEAR_SCREEN` | 2 | Wipe the buffer | none |
| `WRITE_STRING` | 3 | Place a string at coordinates | row(u8), col(u8), attr(u8), ASCIIZ string ≤30 chars |
| `DRAW_SCREEN` | 4 | Commit the buffered frame to the display | none |
| `OPTIONS` | 5 | Set resolution (not used by BF — VTX picks resolution) | resolution mode (u8) |
| `SYS` | 6 | Position a system-rendered element | row(u8), col(u8), element_id(u8) |

## WRITE_STRING attribute byte

```
bit 7 : version (must be 0)
bit 6 : DISPLAYPORT_MSP_ATTR_BLINK (auto-blink toggle)
bits 5-2 : reserved
bits 1-0 : font number (0..3 — four 256-char fonts)
```

(Source: [[betaflight-displayport-api]])

## SYS sub-command

`MSP_DP_SYS` does **not** carry text. It tells the goggles to render one of
its own system-managed elements (voltage, bitrate, RSSI, warnings) at the
given position. This lets the goggles use higher-fidelity assets — bar
graphs, multi-line warnings, RF link icons — that wouldn't fit in a
character-cell font.

## Repaint Cadence

Betaflight sends OSD updates roughly every 25–33 ms (~30 Hz). The typical
sequence per frame:

1. `CLEAR_SCREEN`
2. One or more `WRITE_STRING` calls for each visible OSD element
3. `DRAW_SCREEN` to commit
4. `HEARTBEAT` if no element changed for too long

If the FC stops sending for ~1 second, most digital VTXs show a
"DISCONNECTED" banner.

## Wiring

To enable MSP DisplayPort:

```
# CLI
set osd_displayport_device = MSP
# Ports tab: assign UART to "VTX (MSP + DisplayPort)"
```

(Source: [[betaflight-displayport-api]])

## Why Strings, Not Pixels

Pushing a full framebuffer over UART would saturate the serial link.
Character-positioning is `~30 bytes × N_elements × 30 Hz` — easily fits in
115200–921600 baud. Pixel-perfect rendering happens downstream in the
display device.

## Implementation in BF

- `src/main/io/displayport_msp.c` — driver that emits these MSP commands.
- `src/main/osd/osd.c` calls into the displayport abstraction; the same OSD
  code drives either MAX7456 or MSP DisplayPort.

## Sources

- [[betaflight-displayport-api]]
- [[betaflight-msp-protocol-h]]

## Gaps

> [!gap] Per-VTX font support
> Each digital VTX brand ships its own font(s). The bits-0-1 font selector
> assumes the receiver knows which physical font is loaded as font 0/1/2/3.
> The mapping is vendor-specific and not documented in the BF API page.
