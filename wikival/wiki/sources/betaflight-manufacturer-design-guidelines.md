---
type: source
title: "Betaflight Manufacturer Design Guidelines"
source_type: official-docs
status: ingested
source_url: https://betaflight.com/docs/development/manufacturer/manufacturer-design-guidelines
author: Betaflight Project
confidence: high
created: 2026-05-12
updated: 2026-05-12
tags: [source, official-docs, manufacturer, design-guidelines, mcu-policy]
---

# Betaflight Manufacturer Design Guidelines

## Summary

The **official Betaflight policy document** for FC hardware
manufacturers. Lays out which MCUs to use, peripheral expectations,
gyro choices, and design constraints. This is the closest thing the BF
project has to an authoritative "what is supported" statement.

## Key Claims Extracted

- **Recommended for new designs**: STM32 H7 series (H56x, H72x, H73x,
  H74x).
- **Supported**: STM32 F7X2, STM32 G4XX, STM32 F405 ("older design,
  limited to 4 motors"), AT32F435.
- **Deprecated for new designs**: STM32F411 (explicit:
  *"Betaflight has deprecated implementation of new STM32F411 designs"*).
- **Motor-output cap**: *"Effective immediately, new flight controller
  designs that use the STM F4 and F7 series MCUs will be limited to 4
  motor outputs"* — H7 required for >4 motors.
- **Gyro**: ICM-42688-P is the **recommended** gyro. BMI-270 is
  **discouraged** due to uncalibrated gyroscope behavior. Gyros must be
  SPI; I2C gyros are not supported.
- **I2C peripherals**: 800 kHz default, 4.7 kΩ pull-ups.
- **Blackbox**: at least 8 MB flash chip.
- **LEDs**: at minimum blue; green preferred.

## What It Contributes to the Wiki

The authoritative policy citations for every MCU-status claim in the
vault. Used by:

- [[STM32F722]]
- [[STM32 MCU Family in Betaflight]]
- [[Cloud Build System]]
- [[Betaflight MCU Targets]]

## Confidence

**high** — this is the project's own official document. It is the
authoritative source on what BF supports for new designs.

## Caveats

- The document changes over time as the project's policies evolve. A
  snapshot fetched on a given date should be treated as point-in-time.
  The motor-output cap claim, in particular, was added relatively
  recently (late 2024 per community accounts).

## Provenance

Fetched 2026-05-12.
