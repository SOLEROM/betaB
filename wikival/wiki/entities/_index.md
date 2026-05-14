---
type: index
title: "Entities Index"
created: 2026-05-12
updated: 2026-05-12
tags: [index, entities]
---

# Entities

People, hardware, organizations, and tools in the Betaflight ecosystem.

## Key People

| Name | Role | Page |
|------|------|------|
| Boris B | Betaflight founder | [[Boris B]] |
| Mikael Argall | Core dev (rates, filtering) | stub |
| SupaflyFPV | BF team, tuning | stub |

## Flight Controllers (Hardware)

| FC | MCU | Notable | Page |
|----|-----|---------|------|
| Aocoda F460 Stack | STM32F405 | F405 v2 + 60A ESC | [[Aocoda F460 Stack]] |
| SpeedyBee F405 V3 BLS 50A Stack | STM32F405 | 30×30 F4 + BLS 50A — popular reference | [[SpeedyBee F405 V3 BLS 50A Stack]] |
| Kakute F7 | STM32F7 | Holybro, popular | stub |
| Matek F405 | STM32F4 | Reliable, many variants | stub |
| SpeedyBee F7 | STM32F7 | Budget, popular | stub |
| HGLRC F7 | STM32F7 | Racing | stub |

## On-FC Chips

| Chip | Class | Vendor | Page |
|------|-------|--------|------|
| [[MPU6000]] | IMU (gyro+accel) | Invensense | **page exists** |
| [[BMI270]] | IMU (gyro+accel) | Bosch | **page exists** |
| [[ICM-42688-P]] | IMU (gyro+accel) | Invensense | **page exists** |
| [[MAX7456]] | OSD (analog video) | Maxim/AD | **page exists** |
| [[BMP280]] | Barometer | Bosch | **page exists** |
| [[STM32F722]] | MCU | STM | **page exists** |

## External Modules

| Module | Class | Page |
|--------|-------|------|
| [[uBlox GPS Module]] | GNSS receiver | **page exists** |

## ESC/Motor Ecosystem

| Entity | Type | Notes | Page |
|--------|------|-------|------|
| [[DSHOT]] | ESC Protocol | Digital, primary protocol for BF | wiki/features |
| BLHeli_32 | ESC firmware | Closed-source, supports DSHOT | stub |
| AM32 | ESC firmware | Open-source alternative to BLHeli_32 | stub |
| KISS | ESC/FC ecosystem | Competitor to BF | stub |

## Organizations

| Org | Role | Page |
|-----|------|------|
| Betaflight team | Core maintainers | stub |
| Team Blacksheep (TBS) | CRSF protocol, Crossfire | stub |
| [[ExpressLRS]] | ELRS open-source RC link | **page exists** |
| Holybro | FC/ESC manufacturer | stub |

## Tools

| Tool | Purpose | Page |
|------|---------|------|
| [[Betaflight Configurator]] | GUI config | wiki/configurator |
| Blackbox Explorer | Log analysis | stub |
| betaflight-tx-lua-scripts | TX integration | stub |
| EmuFlight | BF fork with different defaults | stub |
