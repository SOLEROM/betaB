---
type: index
title: "Architecture Index"
created: 2026-05-12
updated: 2026-05-12
tags: [index, architecture]
---

# Architecture

Betaflight codebase structure, key modules, and how they relate.

## Repository Layout

```
betaflight/
├── src/main/
│   ├── fc/          # flight controller core: arming, tasks, runtime config
│   ├── flight/      # PID, mixer, imu, failsafe, position control
│   ├── sensors/     # gyro, accel, baro, compass, rangefinder, ESC telemetry
│   ├── io/          # serial, OSD, VTX, LED, GPS, RC
│   ├── drivers/     # hardware drivers (SPI, I2C, UART, DMA, timer)
│   ├── msp/         # MSP protocol implementation
│   ├── config/      # persistent configuration, EEPROM layout
│   ├── scheduler/   # cooperative task scheduler
│   └── target/      # per-FC-target pin assignments and feature flags
├── lib/             # third-party libraries (ChibiOS, CMSIS, etc.)
└── make/            # build system fragments
```

## Stub Pages to Create

| Module | Priority | Notes |
|--------|----------|-------|
| [[FC Core]] | HIGH | fc/ — arming state machine, task dispatch |
| [[PID Module]] | HIGH | flight/pid.c — the main control loop |
| [[IMU Module]] | HIGH | sensors/gyro — gyro init, calibration, DMA read |
| [[Mixer Module]] | MED | flight/mixer.c — motor output calculation |
| [[MSP Module]] | HIGH | msp/ — all MSP command handlers |
| [[Scheduler]] | MED | scheduler/ — task timing, overrun detection |
| [[Config System]] | MED | config/ — EEPROM layout, defaults, pg_ macros |
| [[Target System]] | MED | target/ — unified targets, manufacturer defaults |
| [[OSD Module]] | MED | io/osd — MAX7456, HD OSD |
| [[Serial Module]] | MED | io/serial — port assignment, baud, MSP/CLI routing |
| [[Drivers Layer]] | LOW | SPI/I2C/UART/DMA — hardware abstraction |
| [[Build System]] | MED | Make + custom toolchain, how to build for a target |
