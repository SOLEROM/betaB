---
type: concept
title: "Companion Computer"
status: developing
created: 2026-05-12
updated: 2026-05-12
tags: [concept, autonomy, raspberry-pi, msp, companion-computer]
---

# Companion Computer

## What It Is

A companion computer is a secondary processing unit (typically a Raspberry Pi, Jetson, or similar SBC) mounted on the drone alongside the flight controller. It handles high-level autonomy — computer vision, mission planning, waypoint navigation — while the flight controller (running Betaflight) handles low-level stabilization, filtering, and safety.

The two communicate over serial or USB using the [[MSP Protocol]].

## Role in Betaflight Autonomous Systems

In a BF-based autonomous drone:

```
Companion Computer (RPi)
    ↓ MSP_SET_RAW_RC
Flight Controller (Betaflight)
    ↓ PID loop
ESCs → Motors
```

The companion computer **does not** replace the FC's control loop — it only overrides the RC input layer via [[MSP Override Mode]]. All of Betaflight's stabilization, filtering, and safety features remain active.

## Typical Architecture

1. **Perception layer** — camera, sensors → companion computer
2. **Decision layer** — mission logic, path planning → companion computer
3. **Command layer** — `MSP_SET_RAW_RC` frames → FC
4. **Control layer** — PID, filters, mixer → FC → ESC
5. **Safety layer** — failsafe, arming, angle limits → FC

## Multi-Threaded Python Pattern

From [[FPV Autonomous Operation with Betaflight and Raspberry Pi]], a minimal companion computer autopilot uses three threads:

| Thread | Role |
|--------|------|
| Router | Dequeues commands and dispatches them |
| Telemetry | Polls MSP_ANALOG + MSP_ALTITUDE; triggers safety responses |
| Pilot | Executes autonomous flight sequences |

## Key MSP Commands for Companion Computers

| Command | Purpose |
|---------|---------|
| `MSP_SET_RAW_RC` | Send synthetic stick values to FC |
| `MSP_RC` | Read current RC channel values |
| `MSP_ALTITUDE` | Read barometer altitude |
| `MSP_ANALOG` | Read battery state |

See [[MSP Protocol]] for full command details.

## Hardware Example

[[Aocoda F460 Stack]] (F405 v2 FC) + Raspberry Pi over USB Type-C.
`msp_override_channels_mask = 47` to override Roll/Pitch/Throttle/Yaw/AUX2.

## Related

- [[MSP Protocol]]
- [[MSP Override Mode]]
- [[Aocoda F460 Stack]]

## Gaps

> [!gap] Jetson / more capable SBCs
> The article only covers RPi. Integration patterns for Jetson Nano (for CV/targeting) not yet documented.

## Sources

- [[FPV Autonomous Operation with Betaflight and Raspberry Pi]]
