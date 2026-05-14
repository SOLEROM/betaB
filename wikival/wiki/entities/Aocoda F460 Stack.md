---
type: entity
title: "Aocoda F460 Stack"
entity_type: hardware
status: active
url: ""
created: 2026-05-12
updated: 2026-05-12
tags: [entity, hardware, flight-controller, esc, f405]
---

# Aocoda F460 Stack

## What It Is

A combined Flight Controller + ESC hardware stack for FPV drones. The "F460" designation refers to the F4-series MCU (STM32F405) and the 60A ESC rating.

| Component | Spec |
|-----------|------|
| Flight Controller | Aocoda RC F405 v2 |
| MCU | STM32F405 |
| ESC | 4-in-1, 60A per motor |
| USB | Type-C (MSP-capable) |
| Betaflight target | Likely `AOCODA_F405` (unverified) |

## Role in Betaflight Ecosystem

Used as the flight controller in a companion-computer autonomous drone build. Connected to Raspberry Pi via USB Type-C with MSP protocol enabled. Runs Betaflight with [[MSP Override Mode]] active.

## Key Facts

| Fact | Value |
|------|-------|
| MCU | STM32F405 |
| ESC rating | 60A (4-in-1) |
| USB | Type-C |
| Protocol | MSP over USB serial |
| Firmware | Betaflight |
| Servo output | SERVO1 on AUX2 (for drop mechanisms) |

## Related

- [[MSP Protocol]]
- [[MSP Override Mode]]
- [[Companion Computer]]

## Sources

- [[FPV Autonomous Operation with Betaflight and Raspberry Pi]]
