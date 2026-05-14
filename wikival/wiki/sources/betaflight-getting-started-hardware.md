---
type: source
title: "Betaflight Getting Started Hardware"
source_type: official-docs
status: ingested
source_url: https://betaflight.com/docs/wiki/getting-started/hardware
author: Betaflight Project
confidence: high
created: 2026-05-12
updated: 2026-05-12
tags: [source, official-docs, getting-started, hardware, mcu, sensors, osd, gps]
---

# Betaflight Getting Started Hardware

## Summary

Official Betaflight getting-started page on flight controller hardware,
covering all of the on-board functional blocks an FC contains: MCU,
voltage regulator, OSD chip, IMU (gyro + accelerometer), barometer,
magnetometer, and GPS. Aimed at end users (not manufacturers; see
[[Betaflight Manufacturer Design Guidelines]] for that audience).

This is the canonical "what's inside a flight controller" reference —
every block in this page maps to a hardware peripheral the BF firmware
talks to via a dedicated driver in `src/main/drivers/`.

## Key Claims Extracted

### MCU (the "brain")
- Almost all FCs use **ARM Cortex-based MCUs**.
- Primary vendor: **STMicroelectronics (STM)**; newer alternative:
  **ArteryTek**.
- Performance tier list (low → high): F411 → G473 → F405 / AT32F435 →
  F722 → F745 → H743.
- General rule cited verbatim: *"the higher the number, the more
  powerful the MCU is."*
- The naming convention encodes flash, RAM, and clock speed.

### Voltage Regulator
- Three rails on a typical FC: **3.3V** (MCU, sensors), **5V**
  (receivers, analog cameras, low-power VTX), and **9V/10V/12V**
  (high-power digital VTX — **12V recommended**).
- Overloading a regulator can cook components; manufacturers spec
  the per-rail current capacity.

### OSD
- **Analog VTX** path: FC has a **MAX7456** (or pin-compatible AT7456E)
  chip. It overlays text/graphics onto the raw camera video before
  the signal goes to the VTX.
- **Digital VTX** path: no on-FC OSD chip. The VTX itself draws the
  overlay in the goggles, which also gives clean recording + OSD
  data export.

### Gyroscope & Accelerometer
- These are usually a single combo chip (the "**IMU**" or "**6-axis**").
- **Gyro** measures angular velocity (used in Acro / rate modes).
- **Accel** measures spatial orientation / gravity (used in Angle,
  Horizon, GPS Rescue).
- Three common parts in 2026:
  1. **Invensense MPU6000** — long-time gold standard; was discontinued
     and recently returned to production.
  2. **Bosch BMI270** — strong alternative; 3.2 kHz max sample rate
     (vs MPU6000's 8 kHz).
  3. **Invensense ICM-42xxx / ICM-20xxx** — adopted during MPU6000
     shortages; since **BF 4.4.1** it performs on par with or better
     than MPU6000.

### Barometer
- Measures atmospheric pressure → altitude.
- Used today for **GPS Rescue landing**; planned to drive altitude
  hold and more advanced flight modes.
- Most common chip: **Bosch BMP280**. Alternates: **BMP180**, **MS5611**,
  **DPS310**.
- Hardware requirement: the package **must have a hole** to feel air
  pressure. Open-cell foam over the hole damps noise without blocking
  airflow. **Conformal coating over the hole kills it.**

### Magnetometer
- Compass — measures Earth's field to recover heading.
- Planned for future GPS Rescue improvements; **not yet fully
  integrated in BF 4.4.1**.
- Hugely sensitive to EMI from motors / ESCs / VTX / RX. External
  units need to be mounted away from the stack and calibrated.

### GPS
- Position + velocity via GNSS (GPS + GLONASS + Galileo).
- Used for: OSD position, crash-localization, GPS Rescue return-to-home.
- Most modules use **uBlox M8**. **M10** is newer and better, with one
  catch:
  - **Before BF 4.5.0**: M10 required manual u-center config; bad
    config → unreliable position fix.
  - **From BF 4.5.0**: BF auto-configures M10 modules; no manual setup.

## What It Contributes to the Wiki

This is the source-of-truth for:

- The official user-facing MCU list (matches [[Betaflight Manufacturer
  Design Guidelines]] with slightly different wording).
- The "higher number = more powerful" framing used across the FPV
  community.
- The canonical FC component inventory (used to scaffold
  [[Flight Controller Hardware]]).
- IMU chip lineage and tradeoffs ([[Inertial Measurement Unit]]).
- OSD chip identity for the analog video path ([[MAX7456]]).
- The BF 4.5.0 GPS M10 auto-config milestone.

Cited by:

- [[STM32 MCU Family in Betaflight]]
- [[STM32F722]]
- [[Flight Controller Hardware]]
- [[Inertial Measurement Unit]]
- [[OSD (On-Screen Display)]]
- [[Barometer (Altitude Sensing)]]
- [[Magnetometer (Compass)]]
- [[GPS (Position Sensing)]]
- [[FC Voltage Rails]]
- [[MPU6000]]
- [[BMI270]]
- [[ICM-42688-P]]
- [[MAX7456]]
- [[BMP280]]
- [[uBlox GPS Module]]

## Confidence

**high** — official BF documentation on the project's main docs site.

## Provenance

Fetched 2026-05-12 from `https://betaflight.com/docs/wiki/getting-started/hardware`.
Raw at `.raw/articles/betaflight-getting-started-hardware-2026-05-12.md`.
