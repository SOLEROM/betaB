---
type: index
title: "Features Index"
created: 2026-05-12
updated: 2026-05-12
tags: [index, features]
---

# Features

One page per major Betaflight feature. Each page covers: what it does, how it's configured, key parameters, and how it interacts with other subsystems.

## Stub Pages to Create

| Feature | Priority | Notes |
|---------|----------|-------|
| [[PID Controller]] | HIGH | Core flight loop — P/I/D gains, feed-forward, iterm_relax |
| [[Rates]] | HIGH | RC rate, super rate, expo — how stick input maps to rotation |
| [[Gyro Filtering]] | HIGH | RPM filter, notch filters, lowpass chain |
| [[Flight Modes]] | HIGH | Acro, Angle, Horizon, Air Mode, Turtle Mode |
| [[Blackbox]] | MED | Data logging format, tools, analysis |
| [[OSD]] | MED | On-screen display elements, layout, fonts |
| [[GPS Rescue]] | MED | Return-to-home, altitude hold |
| [[DSHOT]] | MED | ESC protocol — DSHOT150/300/600, bidirectional DSHOT |
| [[RPM Filter]] | HIGH | Uses bidirectional DSHOT to filter motor noise frequencies |
| [[Arming]] | MED | Arm/disarm logic, prearm, runaway takeoff prevention |
| [[ESC Telemetry]] | LOW | ESC temp/current/RPM via telemetry |
| [[VTX Control]] | LOW | Video transmitter power/channel control via SmartAudio/Tramp |
| [[SmartAudio]] | MED | One-wire VTX control protocol — **page exists** |
| [[LED Strip]] | LOW | Addressable LED configuration |
| [[Failsafe]] | MED | What happens when RC signal is lost |
| [[Anti-Gravity]] | MED | iterm boost on throttle changes |
| [[TPA]] | MED | Throttle PID Attenuation |
| [[Launch Control]] | LOW | Drag race launch assist |
| [[MSP Override Mode]] | LOW | External control via MSP — **page exists** |
| [[CRSF_BAUDRATE]] | MED | Compile-time CRSF UART baud (420000 / 416666) + V3 runtime negotiation — **page exists** |
