---
type: entity
title: "SpeedyBee F405 V3 BLS 50A Stack"
entity_type: hardware
status: active
url: ""
created: 2026-05-12
updated: 2026-05-12
tags: [entity, hardware, flight-controller, esc, f405, speedybee, 30x30]
---

# SpeedyBee F405 V3 BLS 50A Stack

## What It Is

A 30×30 mm combined Flight Controller + 4-in-1 ESC stack from SpeedyBee.
The "F405 V3" identifies the FC (third-revision F4-series board, STM32F405
MCU), and "BLS 50A" identifies the ESC firmware family (BLHeli_S) and
per-channel current rating (50 A continuous).

It is one of the most common "default" FC+ESC stacks in mid-2020s FPV
builds — cheap, well-supported in Betaflight, and the standard reference
point for 5"–7" builds that need 6S compatibility.

## Role in Betaflight Ecosystem

- Ships with a Betaflight target (vendor-published; firmware available
  through the Betaflight Configurator's firmware flasher).
- ESC runs BLHeli_S firmware — programmed over the FC via DSHOT passthrough
  ([[BLHeli Configurator]] equivalent workflow).
- 30×30 mounting pattern means the board fits any standard 5"–7" frame
  stack with M3 hardware.

## Key Facts

| Fact | Value |
|------|-------|
| MCU | STM32F405 (F4 family) |
| FC form factor | 30×30 mm |
| ESC | 4-in-1, 50 A per channel continuous |
| ESC firmware | BLHeli_S (BLS) |
| ESC protocol | DSHOT300 / DSHOT600 supported |
| Voltage range | 3S–6S LiPo / 6S Li-Ion |
| USB | Type-C |
| Sensors | Gyro + accelerometer + barometer (typical for V3) |
| UART count | Multiple (typical: 4–6 for VTX, RX, GPS, ESC tele) |
| Notable feature | Integrated BEC, capacitor pad, OSD chip |

## Typical Configuration in Betaflight

- ESC protocol: DSHOT600.
- Motor poles: depends on motor (14 for most 22xx–28xx outrunners).
- UART map (common pattern):
  - UART1: USB / debug
  - UART2: receiver (CRSF for ELRS/Crossfire)
  - UART3 or UART4: VTX (SmartAudio / Tramp)
  - UART5/6: GPS (when fitted)

## Constructive Use-Case Fit

Equally at home in:
- 5" freestyle and racing
- 7" long-range / SAR / mapping ([[Long-Range 7-Inch Class]])
- Cinema 6"–7" platforms with stabilized payloads
- Educational FPV courses (cheap, robust, well-documented)

## Related

- [[Long-Range 7-Inch Class]]
- [[Li-Ion vs LiPo for FPV]]
- [[ExpressLRS]]
- [[SmartAudio]]
- [[DSHOT]]

## Gaps

> [!gap] BF target name
> Whether the official BF target is `SPEEDYBEEF405V3`, a unified target,
> or a manufacturer-shipped custom config is not yet confirmed against
> the Betaflight firmware repo.

## Sources

- [[How to Build a 7-Inch FPV Drone (constructive extract)]]
