---
type: index
title: "Configurator Index"
created: 2026-05-12
updated: 2026-05-12
tags: [index, configurator]
---

# Configurator

Betaflight Configurator is a Chrome/Electron app for configuring BF over USB. This section covers each tab, the CLI, and key workflows.

## Configurator Tabs

| Tab | Purpose | Page |
|-----|---------|------|
| Welcome | Firmware flash, connect | [[Configurator Welcome Tab]] |
| Ports | UART assignment (MSP, GPS, RX, VTX) | [[Ports Tab]] |
| Configuration | Main switches: FC type, ESC protocol, features | [[Configuration Tab]] |
| Power & Battery | Voltage/current sensing, cell count | [[Power Tab]] |
| Failsafe | RC loss behavior | [[Failsafe Tab]] |
| PID Tuning | P/I/D/FF, filter settings, master multiplier | [[PID Tuning Tab]] |
| Rates & Expo | RC rate curves, stick feel | [[Rates Tab]] |
| Receiver | RX protocol, channel map, stick ranges | [[Receiver Tab]] |
| Modes | Flight mode to AUX channel assignments | [[Modes Tab]] |
| Adjustments | In-flight parameter tuning via AUX | [[Adjustments Tab]] |
| Motors | Motor test, ESC protocol test | [[Motors Tab]] |
| OSD | OSD element layout | [[OSD Tab]] |
| Video Transmitter | VTX power/channel | [[VTX Tab]] |
| LED Strip | LED pattern config | [[LED Tab]] |
| Sensors | Live sensor readouts | [[Sensors Tab]] |
| Logging | Blackbox config | [[Logging Tab]] |
| GPS | GPS config, map | [[GPS Tab]] |
| CLI | Direct CLI access | [[CLI Reference]] |

## CLI Reference

The CLI is the most powerful interface. Key commands:
- `dump` — export all settings
- `diff all` — show non-default settings only
- `set <param> <value>` — change a parameter
- `get <param>` — read a parameter
- `save` — persist to EEPROM
- `feature <name>` — enable a feature
- `resource` — view/remap hardware pins

> [!gap] CLI command list
> Need a comprehensive CLI parameter reference filed under [[CLI Reference]].
