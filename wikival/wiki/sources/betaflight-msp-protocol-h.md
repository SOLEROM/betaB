---
type: source
title: "Betaflight Source — msp_protocol.h"
source_type: source-code
author: Betaflight project
date_published: 2025+
url: https://github.com/betaflight/betaflight/blob/master/src/main/msp/msp_protocol.h
confidence: high
created: 2026-05-14
updated: 2026-05-14
tags: [source, msp, command-catalog, betaflight]
key_claims:
  - "MSP v1 command IDs are defined as preprocessor macros in this header"
  - "Identity block: MSP_API_VERSION(1), MSP_FC_VARIANT(2), MSP_FC_VERSION(3), MSP_BOARD_INFO(4), MSP_BUILD_INFO(5)"
  - "Persistence: MSP_EEPROM_WRITE(250), MSP_REBOOT(68), MSP_RESET_CONF(208)"
  - "MSP_V2_FRAME = 255 reserves the V1 wrap-around slot for tunneled V2"
---

# Betaflight Source — msp_protocol.h

## Summary

The canonical command-ID list for Betaflight MSP v1. Each command is a
`#define` mapping a symbolic name to a uint8_t. This is what the parser and
the configurator both reference (configurator imports an equivalent
generated JS table).

## What it contributes

The full enumerated catalog of v1 commands, including the categories the
configurator uses on every connect:

- **Identity** (always queried first): API_VERSION, FC_VARIANT, FC_VERSION,
  BOARD_INFO, BUILD_INFO, UID.
- **Live state**: STATUS, STATUS_EX, RAW_IMU, ATTITUDE, ALTITUDE, ANALOG,
  RC, MOTOR.
- **Config groups** (per Configurator tab): FEATURE_CONFIG, BATTERY_CONFIG,
  PID, PID_ADVANCED, RC_TUNING, FILTER_CONFIG, ADVANCED_CONFIG,
  FAILSAFE_CONFIG, RX_CONFIG, GPS_CONFIG, GPS_RESCUE, OSD_CONFIG,
  VTX_CONFIG, VTXTABLE_BAND, VTXTABLE_POWERLEVEL, MODE_RANGES,
  ADJUSTMENT_RANGES, LED_STRIP_CONFIG, BLACKBOX_CONFIG, BEEPER_CONFIG.
- **Setters**: matching `MSP_SET_*` pairs.
- **System**: EEPROM_WRITE(250), REBOOT(68), RESET_CONF(208),
  SELECT_SETTING(210).
- **Bulk**: MSP_MULTIPLE_MSP(230) packs multiple requests in one frame.
- **OSD**: OSD_CONFIG(84), OSD_CHAR_READ(86), OSD_CHAR_WRITE(87),
  OSD_CANVAS(189), DISPLAYPORT(182).
- **Sentinel**: MSP_V2_FRAME(255) — encapsulates a V2 frame inside V1
  framing for legacy peers.

## Confidence

**High.** Primary source code. IDs verified against the upstream master
branch on 2026-05-14.

## Related

- [[MSP Protocol]]
- [[MSP API Versioning]]
- [[MSP v2 Frame Format]]
