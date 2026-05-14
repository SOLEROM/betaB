---
type: concept
title: "Inertial Measurement Unit"
status: developing
domain: hardware
created: 2026-05-12
updated: 2026-05-12
aliases: [IMU, Gyro, Gyroscope, Accelerometer]
tags: [concept, hardware, sensors, imu, gyro, accelerometer]
---

# Inertial Measurement Unit (IMU)

## Definition

The **IMU** is the sensor chip the PID loop reads on every cycle to
know what the craft is doing. On a Betaflight FC the IMU is almost
always a **6-axis** combo: three gyroscope axes + three accelerometer
axes in a single SPI-attached package.

- **Gyro** → angular velocity (deg/s on each of pitch, roll, yaw).
- **Accel** → linear acceleration (g on each axis); at rest it gives a
  gravity vector that fixes "down."

Without a working gyro, BF will not arm. The gyro is the most
latency-critical sensor on the entire craft.

## Why It Matters For Flight

- **Acro mode** uses **only the gyro**. The pilot commands a rate
  (deg/s), the gyro measures the actual rate, the PID closes the loop.
  No knowledge of orientation in the world.
- **Angle / Horizon mode** fuses **gyro + accel**. The accel provides
  an absolute "down" reference so the firmware knows true level.
  Without the accel these modes can't exist.
- **GPS Rescue** needs **accel** to keep the craft upright while it
  commands a route home.

## Sample Rates

Higher sample rates mean lower-latency PID and finer noise modeling
for filters. Betaflight runs the gyro at the chip's native rate and
the PID loop at a programmable divider of that rate.

| Chip                | Max gyro rate  |
|---------------------|----------------|
| [[MPU6000]]         | 8 kHz          |
| [[BMI270]]          | 3.2 kHz        |
| [[ICM-42688-P]]     | 8 kHz (32 kHz with FIFO) |

The classic BF guidance was "8k/4k" (8 kHz gyro sample, 4 kHz PID
loop). Modern targets often run "3.2k/3.2k" on BMI270 or 8k/8k on
MPU6000/ICM.

## The Three-Chip Era

Betaflight FCs in 2026 ship with one of three IMU chips. Which one is
on the board affects the *defaults* but not the firmware behavior.

1. **[[MPU6000]]** — Invensense (TDK). Long-running gold standard.
   Was discontinued, recently returned to production.
2. **[[BMI270]]** — Bosch. Replaced MPU6000 during the shortage.
   Lower max rate (3.2 kHz) but very clean signal.
3. **[[ICM-42688-P]] / ICM-20xxx** — Invensense newer parts. Adopted
   during MPU6000 shortages. Performance now equals or beats
   MPU6000 **since BF 4.4.1**.

## In Betaflight

- Drivers: `src/main/drivers/accgyro/accgyro_*.c` (one file per chip).
- Selection: at compile time via `target.h` define; or at runtime when
  the board has multiple IMU sockets.
- Calibration: gyro bias auto-calibrates on arming if motionless.
- Accel needs an explicit "calibrate accelerometer" step from the
  Configurator — it's a 6-position bias measurement.
- Gyro filter chain: see (future) [[Filtering Theory]].

> [!key-insight]
> The choice of IMU chip is the single biggest determinant of a
> board's "feel" before any tuning. Two F405 boards with different
> IMUs need different filter defaults out of the box.

## Key Relationships

- Hardware: [[MPU6000]], [[BMI270]], [[ICM-42688-P]],
  [[Flight Controller Hardware]]
- Future concepts: [[PID Theory]], [[Filtering Theory]], [[Gyro Sampling]]
- Modes that need it: [[GPS Rescue]] (gyro + accel), Acro (gyro only)

## Sources

- [[Betaflight Getting Started Hardware]]
