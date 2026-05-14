---
type: concept
title: "OSD (On-Screen Display)"
status: developing
domain: hardware
created: 2026-05-12
updated: 2026-05-12
tags: [concept, osd, video, hardware]
---

# OSD (On-Screen Display)

## Definition

The **OSD** is the text and graphics overlay drawn on top of the live
FPV video feed: battery voltage, current draw, flight time, RSSI, GPS
distance, crosshair, warnings. Pilots fly by what the OSD tells them.

There are **two architectures** depending on whether the build is
analog or digital video — and they share almost nothing in common at
the hardware level.

## Analog OSD (MAX7456 path)

```
Camera ─► [MAX7456 on FC] ─► VTX ─► RF ─► Goggles
            ▲
            │ SPI (chars, attributes)
          MCU
```

- The flight controller carries a **[[MAX7456]]** chip (or the
  pin-compatible **AT7456E** Chinese equivalent).
- MAX7456 sits in the analog video path *between* the camera and the
  VTX input. It composites a character grid (16×16 px tiles) onto the
  analog signal in real time.
- The MCU pushes character positions and attributes to the MAX7456
  over **SPI**.
- The video that reaches the VTX (and therefore is recorded in the
  goggles) already has the overlay baked in. There is no clean feed.

## Digital OSD (no FC chip)

```
Camera ─► VTX (digital) ─► RF ─► Goggles
                ▲
                │ MSP-DisplayPort (UART)
              MCU
```

- No on-FC OSD chip is needed.
- The MCU sends a high-level draw command stream to the digital VTX
  (DJI O3 / Air Unit, Walksnail Avatar, HDZero) over UART using the
  **MSP DisplayPort** protocol.
- The VTX draws the overlay on the goggle side. This means:
  - **Clean DVR**: the recorded video can omit the overlay if the
    pilot wants.
  - **OSD data export**: telemetry can be re-rendered post-flight.
  - **Better resolution and font flexibility** than MAX7456's fixed
    character set.

## In Betaflight

- The OSD feature lives at `src/main/io/osd.c` (and `src/main/osd/`).
- The MAX7456 driver is `src/main/drivers/max7456.c`.
- The same OSD layout file works for both analog and digital — BF
  abstracts the draw target.
- The OSD is configured via:
  - the **Configurator OSD tab** (drag-to-position),
  - or `set osd_*` CLI commands for raw control.

## Key Relationships

- Hardware: [[MAX7456]], [[Flight Controller Hardware]]
- Related concept: digital VTX systems (DJI O3, Walksnail, HDZero)
- Protocol: MSP DisplayPort (a subset of [[MSP Protocol]])
- Code: `src/main/io/osd.c`, `src/main/drivers/max7456.c`

> [!key-insight]
> The on-FC MAX7456 only exists for analog video. Digital builds skip
> the chip entirely and use MSP DisplayPort over UART instead.

## Sources

- [[Betaflight Getting Started Hardware]]
