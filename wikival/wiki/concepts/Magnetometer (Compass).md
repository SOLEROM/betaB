---
type: concept
title: "Magnetometer (Compass)"
status: developing
domain: hardware
created: 2026-05-12
updated: 2026-05-12
tags: [concept, hardware, sensors, magnetometer, compass, gps-rescue]
---

# Magnetometer (Compass)

## Definition

A **magnetometer** measures the local geomagnetic field vector. Combined
with the accelerometer's gravity vector, it yields **absolute heading**
(yaw) in the world frame — i.e. a compass.

This is the only on-FC sensor that gives true heading. The gyro gives
*rate* of yaw (which drifts when integrated). The GPS gives heading
only when the craft is moving.

## In Betaflight (Status as of BF 4.4.1)

> [!note] Not yet fully integrated
> The official BF docs explicitly call out that the magnetometer is
> **planned to aid advanced GPS Rescue features in future releases**
> but is **not yet fully integrated in 4.4.1**.

When the future integration ships, the practical wins are:

- **Better GPS Rescue heading**, especially at the start of the rescue
  when the craft is near-stationary and GPS heading is unreliable.
- **Yaw drift correction** during slow flight (current Acro yaw drifts
  on a long mission).
- Potential **return-to-home** with proper heading hold.

## Common Chips

- HMC5883L (older, common but discontinued by Honeywell)
- QMC5883L (clone, often labelled as HMC5883L)
- LIS3MDL (ST)
- IST8310 (often paired with M8N GPS modules)
- MMC5983MA (newer, higher resolution)

Driver tree: `src/main/drivers/compass/`. Most are I²C-attached.

## The EMI Problem

This is the single hardest sensor to deploy correctly on a multirotor:

- **Motors** produce huge time-varying magnetic fields (each phase
  switches kHz-rate). Even still motors at idle bias the reading.
- **High-current battery lead** produces a steady offset proportional
  to throttle.
- **VTX / RX antennas** add RF noise.
- The FC ground plane itself can carry mA-level currents that bias
  the reading at the chip.

**Mitigations:**

1. Mount the magnetometer **externally on a mast** above the stack
   (typical: combined with the GPS module).
2. **Calibrate** in place — Configurator has a "rotate the craft in
   all axes" procedure that fits an ellipsoid to the raw readings.
3. Re-calibrate **after any hardware change** (new motors, different
   VTX, longer leads).

## Key Relationships

- Hardware: [[Flight Controller Hardware]]
- Related sensors: [[GPS (Position Sensing)]] (usually on the same
  external module)
- Feature that will use it: [[GPS Rescue]]

> [!key-insight]
> A compass that lives anywhere on the FC stack is mostly measuring
> the stack's own current. Externally mounted is the only way to get
> usable data.

## Sources

- [[Betaflight Getting Started Hardware]]
