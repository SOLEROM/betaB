---
type: concept
title: "GPS (Position Sensing)"
status: developing
domain: hardware
created: 2026-05-12
updated: 2026-05-12
aliases: [GNSS]
tags: [concept, hardware, sensors, gps, gnss]
---

# GPS (Position Sensing)

## Definition

A **GPS** module on a BF craft is an external sensor (not on the FC
PCB) that gives **absolute position and velocity** from a GNSS
constellation. "GPS" is the colloquial term; modern modules listen to
**GPS + GLONASS + Galileo** simultaneously for faster fix and better
accuracy.

The GPS module is its own MCU + RF front end + patch antenna in a
small puck, usually mounted on a mast above the stack along with the
[[Magnetometer (Compass)]].

## In Betaflight — What It Enables

- **OSD position display** — distance to home, speed, sat count.
- **Crash localization** — last known coordinates in OSD/logs.
- **[[GPS Rescue]]** — return-to-home behavior on RC link loss.

GPS does **not** drive flight control directly. The gyro flies the
craft; GPS gives the firmware a slow, absolute position reference
layered on top.

## Module Compatibility — uBlox M8 vs M10

Most BF-compatible GPS modules use **u-blox** silicon.

| Module | Status | Notes |
|--------|--------|-------|
| [[uBlox GPS Module|uBlox M8]] (M8N, M8Q) | Widespread default | "Just works" |
| [[uBlox GPS Module|uBlox M10]] | Newer, better | See caveat below |

> [!warning] M10 quirk before BF 4.5.0
> M10 modules **required manual u-center configuration** under BF
> < 4.5.0. Without it, fixes were slow or unreliable.
> **From BF 4.5.0 onwards** BF auto-configures M10 modules, so manual
> setup is no longer necessary.

## Connection

- **Bus**: UART (3.3 V TTL, asynchronous serial).
- **Default rate**: 9600 baud on first power-up; BF reconfigures to
  the configured baud (usually 38400 or 115200).
- **Protocol**: uBlox UBX binary (BF's default) or NMEA (legacy).

Driver tree: `src/main/io/gps*.c`. The active driver is
`gps_ublox.c`.

## CLI / Config Hooks

- `set gps_provider = UBLOX`
- `set gps_auto_baud = ON`
- `set gps_auto_config = ON` ← this is what triggers M10 auto-setup
- `set gps_rescue_*` family configures rescue behavior

## Key Relationships

- Hardware: [[uBlox GPS Module]], [[Flight Controller Hardware]]
- Co-mounted: [[Magnetometer (Compass)]]
- Feature: [[GPS Rescue]]
- Affects: [[Long-Range 7-Inch Class]] (cruisers depend on GPS)

> [!key-insight]
> If a long-range build "gets bad fix" on BF < 4.5.0 with an M10
> module, the firmware never configured the chip. Upgrade to 4.5+
> or run a manual u-center session.

## Sources

- [[Betaflight Getting Started Hardware]]
