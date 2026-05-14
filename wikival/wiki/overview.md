---
type: overview
title: "Betaflight Overview"
status: stub
created: 2026-05-12
updated: 2026-05-12
tags: [betaflight, overview]
---

# Betaflight Overview

> [!key-insight] What is Betaflight?
> Betaflight is an open-source flight controller firmware for RC multirotor aircraft (drones). It runs on STM32-based flight controller (FC) hardware and is the dominant firmware for FPV racing and freestyle drones.

## Origin

- Forked from Cleanflight in 2015 by Boris B
- Cleanflight itself forked from Baseflight (first major STM32 FC firmware)
- Lineage: MultiWii → Baseflight → Cleanflight → Betaflight

## What It Does

Betaflight reads sensor data (gyroscope, accelerometer, barometer, GPS, RC receiver) and runs a PID control loop to stabilize and control a drone. It outputs motor commands via ESC protocols (DSHOT, OneShot, Multishot, PWM).

## Key Capabilities

- PID flight stabilization (Acro, Angle, Horizon modes)
- Advanced gyro filtering (RPM filter, notch filters, PT1/PT2/PT3 lowpass)
- Blackbox flight data logging
- OSD (On-Screen Display) overlay for FPV goggles
- GPS rescue / Return-to-Home
- CLI (Command Line Interface) for deep configuration
- Betaflight Configurator (cross-platform GUI)
- MSP (MultiWii Serial Protocol) for communication
- Multiple receiver protocols: CRSF, SBUS, IBUS, ELRS, SUMD, SRXL

## Repository

- GitHub: `betaflight/betaflight`
- Language: C (with some C++)
- Build system: Make + custom toolchain (arm-none-eabi-gcc)
- Targets: 400+ FC hardware targets

## Current State (as of 2026)

> [!gap] Version currency
> Need to verify current stable release version and major changes post-4.4.

## Canvases

- [[map.canvas|Wiki Map]] — visual map of all 33 wiki pages grouped by domain (hardware, MCU, sensors, protocols, reverse engineering, synthesis).
- [[main.canvas|Main]] — default scratch canvas for images, PDFs, and ad-hoc notes.

## Related Entities

- [[Betaflight Configurator]] — the GUI tool
- [[MSP Protocol]] — communication protocol
- [[STM32]] — the MCU family BF runs on
- [[DSHOT]] — the dominant ESC protocol

## See Also

- [[Research Overview]] — synthesis of what we know
- [[Open Questions]] — gaps to investigate
