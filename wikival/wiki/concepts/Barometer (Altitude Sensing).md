---
type: concept
title: "Barometer (Altitude Sensing)"
status: developing
domain: hardware
created: 2026-05-12
updated: 2026-05-12
tags: [concept, hardware, sensors, barometer, altitude]
---

# Barometer (Altitude Sensing)

## Definition

A **barometer** on an FC is a MEMS pressure sensor that reads ambient
atmospheric pressure. Pressure changes monotonically with altitude
(about 12 Pa per meter near sea level), so the firmware can convert
pressure → relative altitude.

A barometer gives BF an **absolute vertical reference** that the IMU
alone cannot provide.

## In Betaflight

### Current use (BF 4.4.x)
- **[[GPS Rescue]] landing phase** — the baro tells the craft when
  it is near ground so the descent can be smoothed.
- **OSD altitude readout** — when present, it is more accurate than
  GPS altitude.

### Planned
- **Altitude hold mode** (throttle = altitude rate)
- More sophisticated GPS Rescue trajectories
- Combined baro+GPS+IMU vertical state estimator

## Common Chips

| Chip                | Vendor | Notes                                |
|---------------------|--------|--------------------------------------|
| **[[BMP280]]**      | Bosch  | Most widespread on BF FCs            |
| BMP180              | Bosch  | Older; less common today             |
| MS5611              | TE     | High accuracy, larger package        |
| DPS310              | Infineon | Used on some newer boards          |

Driver tree: `src/main/drivers/barometer/`.

## Physical Constraints (the hole problem)

A barometer is a sealed MEMS die with **one tiny port** that has to
see ambient air. This causes three common installation failures:

1. **Conformal coating** sprayed over the hole **destroys the
   barometer**. Hardware fix: mask the chip during coating.
2. **Direct prop-wash or jet of air** across the port gives noisy /
   wrong readings (the dynamic pressure shows up as a pressure swing).
   Software fix: filtering. Hardware fix: open-cell foam over the
   chip.
3. **Sun heating the chip** → small pressure offsets from thermal
   gradient inside the package. The firmware drift-compensates with
   accel data over short windows.

The official BF docs call out points 1 and 2 explicitly.

## Key Relationships

- Hardware: [[BMP280]], [[Flight Controller Hardware]]
- Feature that depends on it: [[GPS Rescue]]
- Bus: usually **I²C** (sometimes SPI on H7 boards)

> [!key-insight]
> Conformal-coat the rest of the FC but not the baro. Open-cell foam
> over the chip is the standard mitigation against prop wash.

## Sources

- [[Betaflight Getting Started Hardware]]
