---
type: entity
title: "uBlox GPS Module"
entity_type: hardware
status: active
url: https://www.u-blox.com/en/positioning-chips-and-modules
created: 2026-05-12
updated: 2026-05-12
aliases: [u-blox, M8N, M8Q, M10]
tags: [entity, hardware, gps, gnss, ublox]
---

# uBlox GPS Module

## What It Is

**u-blox AG** is the Swiss vendor whose GNSS receiver chips power
the vast majority of GPS modules sold for FPV drones (and a lot of
the rest of the embedded GNSS world). On Betaflight craft they appear
as external puck-style modules — chip + crystal + antenna + casing —
connected to the FC via UART.

## Role in Betaflight Ecosystem

The default GPS provider in Betaflight. The driver speaks **UBX**
(u-blox's binary protocol) over UART. Auto-configuration includes
setting the message rates, output format, and (on M10) the GNSS
constellations to use.

Driver: `src/main/io/gps_ublox.c`. Higher layer: `src/main/io/gps.c`.

## Generations on FPV Modules

| Gen | Common module | Status | Notes |
|-----|---------------|--------|-------|
| M8  | Beitian M8N, BN-880 | Workhorse | "Just works" on any BF version |
| M10 | Beitian M10, Holybro M10 | New, better | **Needs BF 4.5.0+ for auto-config** |

### The M10 caveat (verbatim from BF docs)

> Pre-4.5.0 Betaflight: M10 modules required manual u-center
> configuration; improper setup caused unreliable performance.
> From 4.5.0 onwards: M10 modules receive automatic configuration
> support; manual setup is no longer necessary.

If you bought an M10 module to fly with BF < 4.5.0 and got a bad
fix, the firmware never told the chip to start emitting useful
messages.

## Key Facts

| Fact | Value |
|------|-------|
| Vendor | u-blox AG (Switzerland) |
| Common protocols | **UBX** (BF default), NMEA 0183 |
| Common UART baud | 9600 default; 38400/115200 after BF auto-config |
| Constellations | GPS, GLONASS, Galileo, BeiDou (M10) |
| BF status | Tier-1, default `gps_provider = UBLOX` |
| External module form factor | Puck + mast, integrates with magnetometer often |

## Combined GPS+Mag Modules

Many FPV GPS modules ship with an integrated [[Magnetometer (Compass)|magnetometer]]
chip (commonly IST8310 or QMC5883L) on the same I²C tap so a single
external puck delivers both position and heading.

## Related

- [[GPS (Position Sensing)]]
- [[Magnetometer (Compass)]]
- [[GPS Rescue]]
- [[Flight Controller Hardware]]

## Sources

- [[Betaflight Getting Started Hardware]]
