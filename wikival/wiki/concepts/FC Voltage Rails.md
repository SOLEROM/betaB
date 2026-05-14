---
type: concept
title: "FC Voltage Rails"
status: developing
domain: hardware
created: 2026-05-12
updated: 2026-05-12
tags: [concept, hardware, power, voltage-regulator]
---

# FC Voltage Rails

## Definition

A modern Betaflight flight controller takes raw **pack voltage**
(roughly 11–25 V depending on 3S / 4S / 6S) and generates **three
regulated rails** for the on-board chips and external peripherals.
Each rail has a current capacity set by the FC's regulator IC.

## The Three Rails

| Rail   | Who uses it                                              |
|--------|----------------------------------------------------------|
| **3.3 V** | MCU, gyro/accel, baro, mag, OSD chip, GPS module digital side |
| **5 V**   | RC receivers, analog FPV cameras, low-power VTXs, buzzers, LEDs |
| **9 / 10 / 12 V** | High-power digital VTXs. **12 V is recommended** when present. |

Not every FC carries the 9–12 V rail; it is a feature found on boards
designed for digital video systems (DJI O3, Walksnail, HDZero).

## Why Three Levels

- **3.3 V** — the MCU's native I/O voltage. Sensors that share the
  MCU's SPI/I²C bus must be 3.3 V to avoid level shifting.
- **5 V** — historical "TTL" rail. Almost every RC receiver and
  analog VTX expects 5 V on its supply pin.
- **9–12 V** — digital VTX modules pull serious current (1–3 A
  bursts); a buck from pack voltage to 12 V is far more efficient
  than dropping pack voltage all the way to 5 V.

## The Overload Failure Mode

The most common dead-FC story isn't bad code — it's a regulator
melting because the pilot connected too much to one rail.

Concrete failure paths:

- 5 V rail with 1 A capacity, pilot plugs in a 1 W analog VTX **and**
  a 600 mA LED strip → 5 V rail collapses or LDO cooks.
- 9 V rail with 2 A capacity, pilot connects a DJI Air Unit **and**
  the GPS module → DJI's bootup surge browns out the GPS.
- Reversed polarity on a JST connector → instant regulator death.

The fix is to check the FC manufacturer's spec sheet for **per-rail
current** before wiring, and to use a separate BEC for big loads.

## In Betaflight

The firmware does not directly control the regulators (they are
hardware-only LDOs/bucks), but it **monitors** several voltages:

- **Battery voltage** via a divider into an ADC pin (`set vbat_scale`)
- **Current draw** via a hall-effect or shunt sensor (`set ibata_scale`)
- The OSD reports both; the failsafe triggers on low voltage.

Driver: `src/main/sensors/voltage.c`, `src/main/sensors/current.c`.

## Key Relationships

- Block diagram: [[Flight Controller Hardware]]
- Battery context: [[Li-Ion vs LiPo for FPV]]
- Related feature: Voltage / Current Monitoring (OSD)

> [!key-insight]
> A "blown FC" is usually a blown regulator. Match peripheral current
> draw to the spec for each rail; don't power a high-draw light strip
> off the same rail as the receiver.

## Sources

- [[Betaflight Getting Started Hardware]]
