---
type: feature
title: "SmartAudio"
status: developing
area: vtx
since_version: ""
created: 2026-05-12
updated: 2026-05-12
tags: [feature, vtx, smartaudio, uart]
---

# SmartAudio

## What It Does

SmartAudio is a one-wire serial protocol (originated by TBS) that lets
the flight controller configure a video transmitter at runtime. With
SmartAudio active, the pilot can change VTX **band**, **channel**, and
**output power** from the OSD menu or from radio switches — without
landing or plugging anything in.

It's also the path that enables one of the most important on-ground
safety behaviors in modern Betaflight builds: **VTX automatically drops
to low power when the FC is disarmed**.

## How to Configure

### Wiring

SmartAudio is a single-wire protocol. The VTX's "SmartAudio" pin
connects to the **TX** pin of an unused UART on the FC. (Some VTXs share
the SmartAudio line with their audio-output pad — read the VTX manual.)

### Betaflight Configurator

1. **Ports tab** — on the UART connected to the VTX, set the *Peripherals*
   column to **VTX (TBS SmartAudio)**.
2. **VTX tab** — once the link is up, BF reads the VTX's identity and
   lets you set band, channel, power, and pit-mode behavior directly.
3. **Modes tab** (optional) — assign a radio switch to **VTX PIT MODE**
   for instant power drop on the bench.

### Key Parameters

| Parameter | Default | Range | Notes |
|-----------|---------|-------|-------|
| `vtx_band` | (varies) | 1–5 | A, B, E, F, R bands |
| `vtx_channel` | 1 | 1–8 | Channel within band |
| `vtx_power` | 1 | 1–N | VTX-defined power levels (e.g. 25/200/500/1000 mW) |
| `vtx_low_power_disarm` | OFF | OFF/ON/UNTIL_FIRST_ARM | Drop to lowest power when disarmed |
| `vtx_pit_mode_freq` | 0 | — | Out-of-band frequency for pit mode |

### CLI

```
set vtx_band = 5
set vtx_channel = 1
set vtx_power = 2
set vtx_low_power_disarm = ON
save
```

## How It Works

SmartAudio is a 4800-baud (v1) / 4800–9600 baud (v2) half-duplex serial
protocol. BF's VTX subsystem polls the VTX over the configured UART,
exchanging short framed messages to read the VTX's capability table and
to push new settings.

The BF source side lives roughly in:

- `src/main/io/vtx_smartaudio.c` — the protocol implementation
- `src/main/io/vtx.c` — the chipset-agnostic VTX abstraction
- `src/main/cms/cms_menu_vtx_smartaudio.c` — the OSD menu integration

## Interactions

- **Works with**: [[OSD]] (VTX menu), [[Modes Tab]] (pit-mode switch),
  [[Arming]] (drives `low_power_disarm` transitions).
- **Competes with**: [[Tramp Telemetry]] — IRC Tramp's protocol covers
  the same ground for a different VTX family.
- **Depends on**: a spare UART with TX.

## Constructive Use-Case Fit

- **Racing** — quick band/channel changes between heats without unscrewing
  a thing.
- **Search and rescue** — keep the VTX in pit mode until rotor spin-up,
  switch to full power on takeoff to maximize ground-station range.
- **Cinema** — adjust power per environment (low in tight gimbal shots,
  high for long-range tracking) without landing.
- **Educational** — students can experiment with frequency planning and
  see immediate feedback in the goggles.

## Safety Behaviors Worth Knowing

- **`vtx_low_power_disarm = ON`** drops output to minimum on disarm.
  Reduces ground co-channel interference at the pit and helps prevent
  the VTX from overheating with no airflow.
- **Pit mode** — some VTXs implement a hardware "pit mode" output (e.g.
  ~1 mW or far-out-of-band frequency) that lets multiple pilots arm
  simultaneously without stomping on each other's video.
- **Never power a VTX without an antenna connected.** The unmatched
  output stage will burn. This is a universal FPV rule, not specific to
  SmartAudio, but always worth restating.

## Gaps / Open Questions

> [!gap] SmartAudio v2.1 quirks
> Some VTXs implement v2.1 with extended capability tables; BF's parser
> tolerates them but the exact behavior matrix isn't documented here.

> [!gap] Long-power disarm timing
> Does BF drop power on disarm immediately or on a debounce? Worth a
> source dive.

## Sources

- [[How to Build a 7-Inch FPV Drone (constructive extract)]]
- TBS SmartAudio specification (public)
- Betaflight source — `src/main/io/vtx_smartaudio.c`
