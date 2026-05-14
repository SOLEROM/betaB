---
type: concept
title: "Flight Controller Hardware"
status: developing
domain: hardware
created: 2026-05-12
updated: 2026-05-12
tags: [concept, hardware, fc, architecture]
---

# Flight Controller Hardware

## Definition

A **flight controller (FC)** is a single PCB carrying everything the
firmware ([[Betaflight]]) needs to fly a multirotor: the MCU, sensors,
OSD chip, voltage rails, and the connectors that talk to the rest of
the craft (ESCs, receiver, VTX, camera, GPS).

This page is the inventory of "what's on the board." Each block links
to its own concept or entity page.

## The Block Diagram

```
                          ┌──────────────────┐
        Battery (3S–6S)─┬─►│ Voltage Regulator│─► 3.3V (MCU, sensors)
                        │  │                  │─► 5V  (RX, cam, low-W VTX)
                        │  │                  │─► 9/10/12V (digital VTX)
                        │  └──────────────────┘
                        │           │
                        │           ▼
                        │   ┌──────────────┐
                        │   │     MCU      │◄── Gyro/Accel  (SPI)
                        │   │ STM32 / AT32 │◄── Baro        (I²C)
                        │   │              │◄── Mag         (I²C)
                        │   │              │◄── GPS         (UART)
                        │   │              │◄── RX          (UART/SBUS)
                        │   │              │──► ESCs (4×)   (DSHOT)
                        │   │              │──► VTX control (UART)
                        │   │              │──► OSD (SPI)
                        │   │              │──► USB         (config)
                        │   └──────────────┘
                        │           │
                        │           ▼
                        │   ┌──────────────┐
                   Camera──►│   MAX7456    │──► VTX (analog only)
                        │   │  (OSD chip)  │
                        │   └──────────────┘
                        │
                        └─► ESC power (full pack voltage)
```

## The Blocks

### Compute
- **[[Betaflight MCU Targets|MCU]]** — runs the PID loop, scheduler,
  drivers. Tier list: [[STM32F411]] (deprecated) → STM32G473 →
  [[STM32F722]] / STM32F405 / AT32F435 → STM32F745 → STM32H743.

### Power
- **[[FC Voltage Rails]]** — three rails (3.3 V, 5 V, 9–12 V) generated
  from pack voltage. Over-loading a rail kills the regulator.

### Video / OSD
- **[[OSD (On-Screen Display)]]** — text overlay on FPV video.
  - Analog: [[MAX7456]] chip on the FC.
  - Digital: VTX-side overlay; no on-FC OSD chip.

### Sensors
- **[[Inertial Measurement Unit]]** — gyro + accel combo chip.
  Drivers in `src/main/drivers/accgyro/`.
  - [[MPU6000]], [[BMI270]], [[ICM-42688-P]].
- **[[Barometer (Altitude Sensing)]]** — pressure → altitude.
  - [[BMP280]] (most common), BMP180, MS5611, DPS310.
- **[[Magnetometer (Compass)]]** — heading from Earth's field.
  - Not yet fully integrated as of BF 4.4.1.
- **[[GPS (Position Sensing)]]** — external module, UART-attached.
  - [[uBlox GPS Module]] (M8 / M10).

### IO
- USB (config + DFU bootloader)
- UARTs (RX, GPS, VTX SmartAudio, telemetry)
- Motor outputs (4× [[DSHOT]])
- SPI (gyro, OSD)
- I²C (baro, mag)

## Why This Matters

Every BF feature touches one of these blocks:

| Feature                | Hardware block            |
|------------------------|---------------------------|
| Acro mode              | Gyro                      |
| Angle / Horizon mode   | Gyro + Accel              |
| OSD                    | MAX7456 *or* digital VTX  |
| GPS Rescue             | Accel + Baro + Mag + GPS  |
| Altitude hold (future) | Baro                      |
| Configurator over USB  | USB peripheral on MCU     |
| ESC telemetry          | UART RX from ESC          |
| [[SmartAudio]] / Tramp | UART TX → VTX             |

The BF target file (`src/main/target/<TARGET>/target.h`) is essentially
a declaration of which of these blocks are wired to which MCU pins on
that specific board.

## Key Relationships

- Component-level: [[MCU]], [[OSD (On-Screen Display)]],
  [[Inertial Measurement Unit]], [[Barometer (Altitude Sensing)]],
  [[Magnetometer (Compass)]], [[GPS (Position Sensing)]],
  [[FC Voltage Rails]]
- Reference boards: [[Aocoda F460 Stack]],
  [[SpeedyBee F405 V3 BLS 50A Stack]]
- Build system: [[Cloud Build System]],
  [[ARM Cortex-M Firmware Build Process]]

> [!key-insight]
> A BF "target" is just a mapping from these abstract hardware blocks
> to specific MCU pins. Same firmware core, different `target.h`.

## Sources

- [[Betaflight Getting Started Hardware]]
- [[Betaflight Manufacturer Design Guidelines]]
