---
type: index
title: "Concepts Index"
created: 2026-05-12
updated: 2026-05-12
tags: [index, concepts]
---

# Concepts

Flight control theory and Betaflight-specific technical concepts. These are the "why" behind the features.

## Stub Pages to Create

| Concept | Priority | Notes |
|---------|----------|-------|
| [[PID Theory]] | HIGH | Proportional-Integral-Derivative control loop fundamentals |
| [[Signal Chain]] | HIGH | RC input → rates → PID → mixer → motors |
| [[Gyro Sampling]] | HIGH | IMU sampling rate, loop frequency (8kHz/4kHz/2kHz) |
| [[Filtering Theory]] | HIGH | Lowpass, notch, biquad — tradeoffs of latency vs noise |
| [[Motor Mixing]] | MED | How PID outputs combine for quad/hex/y6 geometries |
| [[ESC Protocols]] | MED | PWM → OneShot → Multishot → DSHOT evolution |
| [[RC Protocols]] | MED | SBUS, IBUS, CRSF, ELRS, SRXL — latency comparison |
| [[Propwash]] | MED | Cause and BF mitigations |
| [[Feed Forward]] | HIGH | Derivative-on-setpoint vs derivative-on-measurement |
| [[iterm Relax]] | MED | Integral windup prevention during flips/rolls |
| [[Blackbox Analysis]] | MED | Reading BF Blackbox data, Blackbox Explorer |
| [[Target System]] | MED | Unified Targets, custom_defaults, manufacturer IDs |
| [[Scheduler]] | LOW | Betaflight's task scheduler, task priorities, cycle times |
| [[Companion Computer]] | MED | RPi/SBC + BF via MSP — **page exists** |
| [[Long-Range 7-Inch Class]] | MED | 7" cruiser configuration pattern — **page exists** |
| [[Li-Ion vs LiPo for FPV]] | MED | Battery chemistry tradeoff + BF threshold implications — **page exists** |
| [[Flight Controller Hardware]] | HIGH | The on-board block inventory of an FC — **page exists** |
| [[OSD (On-Screen Display)]] | HIGH | MAX7456 analog vs digital MSP DisplayPort — **page exists** |
| [[Inertial Measurement Unit]] | HIGH | Gyro + accel combo chip; PID's input sensor — **page exists** |
| [[Barometer (Altitude Sensing)]] | MED | Pressure → altitude; GPS Rescue landing — **page exists** |
| [[Magnetometer (Compass)]] | MED | Heading sensor; EMI-sensitive; not yet integrated in 4.4.1 — **page exists** |
| [[GPS (Position Sensing)]] | MED | uBlox M8/M10; BF 4.5.0 M10 auto-config — **page exists** |
| [[FC Voltage Rails]] | MED | 3.3 V / 5 V / 12 V — overloading = blown FC — **page exists** |
| [[Finding Defines in Firmware Binaries]] | MED | How `#define`s manifest in `.elf`/`.hex`/`.map`; toolchain recipes — **page exists** |
