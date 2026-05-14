---
type: source
title: "FPV Autonomous Operation with Betaflight and Raspberry Pi"
status: ingested
source_url: https://medium.com/illumination/fpv-autonomous-operation-with-betaflight-and-raspberry-pi-0caeb4b3ca69
raw_file: .raw/articles/fpv-autonomous-operation-betaflight-rpi-2026-05-12.md
author: Dmytro Sazonov
published: 2025-02-24
created: 2026-05-12
updated: 2026-05-12
tags: [source, msp, autonomy, raspberry-pi, companion-computer]
---

# FPV Autonomous Operation with Betaflight and Raspberry Pi

**Author:** Dmytro Sazonov | **Published:** 2025-02-24 | **Medium / ILLUMINATION**

## Summary

A developer template for building autonomous FPV combat drones using Betaflight and a Raspberry Pi companion computer. The system uses Betaflight's **MSP OVERRIDE mode** to inject synthetic RC channel values via `MSP_SET_RAW_RC`, enabling autonomous flight while keeping BF's PID loop, filtering, and failsafe active.

Primary use case: drop a payload after flying forward for 2 seconds. Designed as a starting point for Ukrainian drone R&D programs.

## Key Technical Contributions

- Demonstrates `msp_override_channels_mask = 47` to select which channels the companion computer overrides
- Shows a multi-threaded Python autopilot architecture (Router / Telemetry / Pilot threads)
- Documents the four MSP commands needed for basic autonomy: `MSP_ANALOG`, `MSP_ALTITUDE`, `MSP_RC`, `MSP_SET_RAW_RC`
- Hardware: [[Aocoda F460 Stack]] (F405 v2 + 60A ESC) + Raspberry Pi over USB Type-C

## Pages Created from This Source

- [[MSP Protocol]] — MSP command table expanded with autonomy commands
- [[MSP Override Mode]] — feature page for BF's companion-computer override mechanism
- [[Companion Computer]] — concept: using RPi/SBC alongside BF for autonomy
- [[Aocoda F460 Stack]] — entity: hardware used in this build

## Gaps Identified

- Exact Python code not reproduced (GitHub link not confirmed working)
- MSP_SET_RAW_RC payload format (byte order, channel count) not fully specified
- Which BF version introduced MSP OVERRIDE mode is unknown
- Whether ALTHOLD is a standard BF mode or requires GPS/baro addon unclear
