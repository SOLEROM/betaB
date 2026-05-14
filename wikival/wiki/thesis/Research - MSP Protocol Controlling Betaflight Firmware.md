---
type: synthesis
title: "Research: MSP Protocol Controlling Betaflight Firmware"
created: 2026-05-14
updated: 2026-05-14
tags: [research, msp, betaflight, configurator, protocol]
status: developing
related:
  - "[[MSP Protocol]]"
  - "[[MSP v2 Frame Format]]"
  - "[[MSP DisplayPort]]"
  - "[[MSP API Versioning]]"
  - "[[MSP over CRSF]]"
  - "[[MSP Override Mode]]"
  - "[[Companion Computer]]"
sources:
  - "[[inav-wiki-msp-v2]]"
  - "[[betaflight-deepwiki-msp]]"
  - "[[betaflight-displayport-api]]"
  - "[[betaflight-msp-protocol-h]]"
  - "[[betaflight-crsf-protocol-h]]"
---

# Research: MSP Protocol Controlling Betaflight Firmware

## Overview

MSP (MultiWii Serial Protocol) is the **single wire surface** Betaflight
exposes for everything except realtime RC input: configuration, telemetry,
firmware reboot, OSD rendering on digital VTX, wireless config tunneling.
Every Betaflight Configurator interaction is an MSP exchange. This research
pass mapped the wire format (v1 + v2), the full command catalog, the
Configurator handshake, MSP DisplayPort for digital OSD, and the
MSP-over-CRSF wireless tunnel.

## Key Findings

### Wire Format

- **MSP v1**: `$M` preamble, 8-bit command ID, 8-bit size, XOR checksum.
  Max 255-byte payload. Still the default for short commands and legacy
  clients. (Source: [[inav-wiki-msp-v2]])
- **MSP v2**: `$X` preamble, 16-bit command ID, 16-bit size,
  `crc8_dvb_s2` checksum (polynomial `0xD5`). Max 65535-byte payload.
  Backwards compatibility: a V2 frame can be wrapped inside a V1 frame
  with command field set to **255** (`MSP_V2_FRAME`), so V1-only parsers
  ignore it cleanly. (Source: [[MSP v2 Frame Format]])
- MSP v2 IDs are **partitioned by space**:
  `0x0000-0x00FF` mirrors V1 commands, `0x1000+` is `MSP2_COMMON_*`,
  `0x1F00+` is `MSP2_SENSOR_*`, `0x3000+` is `MSP2_BETAFLIGHT_*`.
  (Source: [[betaflight-msp-protocol-h]])

### Command Catalog (V1)

- **Identity**: `MSP_API_VERSION(1)`, `MSP_FC_VARIANT(2)`,
  `MSP_FC_VERSION(3)`, `MSP_BOARD_INFO(4)`, `MSP_BUILD_INFO(5)`,
  `MSP_UID(160)`.
- **Live state**: `MSP_STATUS(101)`, `MSP_RAW_IMU(102)`, `MSP_RC(105)`,
  `MSP_ATTITUDE(108)`, `MSP_ALTITUDE(109)`, `MSP_ANALOG(110)`,
  `MSP_MOTOR(104)`, `MSP_MOTOR_TELEMETRY(139)`.
- **Configuration groups** (one per Configurator tab):
  `MSP_FEATURE_CONFIG`, `MSP_PID`, `MSP_PID_ADVANCED`, `MSP_RC_TUNING`,
  `MSP_FILTER_CONFIG`, `MSP_FAILSAFE_CONFIG`, `MSP_RX_CONFIG`,
  `MSP_GPS_CONFIG`, `MSP_GPS_RESCUE`, `MSP_VTX_CONFIG`,
  `MSP_OSD_CONFIG`, `MSP_MODE_RANGES`, `MSP_ADJUSTMENT_RANGES`,
  `MSP_LED_STRIP_CONFIG`, `MSP_BLACKBOX_CONFIG`. Each has a matching
  `MSP_SET_*`.
- **Persistence**: `MSP_EEPROM_WRITE(250)` writes to flash storage;
  `MSP_REBOOT(68)` resets the MCU; `MSP_RESET_CONF(208)` factory-resets.
- **Bulk**: `MSP_MULTIPLE_MSP(230)` packs multiple commands into one
  frame to cut round-trips.
- **OSD**: `MSP_OSD_CONFIG(84)` for layout, `MSP_OSD_CHAR_READ/WRITE
  (86/87)` for font glyphs, `MSP_DISPLAYPORT(182)` for digital OSD.
  (Source: [[betaflight-msp-protocol-h]])

### Configurator Connect Sequence

```
1. MSP_API_VERSION  → 3 bytes: [protocol_ver, major, minor]
2. MSP_FC_VARIANT   → 4-byte ASCII (BTFL / INAV / CLFL)
3. MSP_FC_VERSION   → 3 bytes: [major, minor, patch]
4. MSP_BUILD_INFO   → 19 bytes: date + time + git hash
5. MSP_BOARD_INFO   → board ID + hw rev + …
6. (capability-gated subsequent queries based on API version)
```

API version is the **breaking-change axis** — every protocol change bumps
`API_VERSION_MINOR` in `version.h`. Recent: BF 4.5 → API 1.46,
BF 4.6 → API 1.47.
(Source: [[MSP API Versioning]], [[betaflight-deepwiki-msp]])

### Save-Settings Flow

Configurator "Save" is `MSP_SET_*` × N + `MSP_EEPROM_WRITE(250)` +
`MSP_REBOOT(68)`. Both `EEPROM_WRITE` and `REBOOT` are aborted if the FC
is armed. EEPROM_WRITE triggers `writeEEPROM()` → `readEEPROM()` and
re-init of the VTX table.
(Source: [[MSP API Versioning]])

### MSP DisplayPort (digital VTX OSD)

- `MSP_DISPLAYPORT(182)` with one of 7 sub-commands.
- Frame loop: `CLEAR_SCREEN(2)` → multiple `WRITE_STRING(3)` → `DRAW_SCREEN(4)`
  at ~30 Hz, plus periodic `HEARTBEAT(0)`.
- Strings are positioned by row/col; goggles render with their own font;
  4 fonts × 256 glyphs selectable per-cell via the attribute byte.
- `MSP_DP_SYS(6)` lets the FC tell the goggles to render its own
  high-fidelity system elements (voltage, bitrate, warnings).
- This is what enables clean DVR, OSD-data export, and goggle-native fonts
  on DJI O3, Walksnail, and HDZero.
  (Source: [[MSP DisplayPort]])

### MSP-over-CRSF (wireless config)

- CRSF frame types `0x7A` (request), `0x7B` (response, 58-byte chunks),
  `0x7C` (write, 8-byte chunks).
- The asymmetric chunk size — 58 down vs 8 up — comes from OpenTX's
  telemetry buffer limit on outbound frames.
- Carries the full MSP byte stream fragmented into CRSF frames; FC side
  reassembles and dispatches through the normal MSP parser.
- Enables Configurator-over-ELRS, including wifi/TCP relay through the
  ExpressLRS TX module.
  (Source: [[MSP over CRSF]])

### Implementation Anchors

| Concern | File |
|---------|------|
| Frame parser | `src/main/msp/msp_serial.c` (`mspSerialProcessReceivedData()`) |
| V1 command IDs | `src/main/msp/msp_protocol.h` |
| V2 Betaflight commands | `src/main/msp/msp_protocol_v2_betaflight.h` |
| V2 common commands | `src/main/msp/msp_protocol_v2_common.h` |
| Generic dispatcher | `src/main/msp/msp.c` → `mspCommonProcessOutCommand()` |
| BF dispatcher | `src/main/msp/msp.c` → `mspFcProcessOutCommand()` |
| RC injection | `src/main/rx/msp.c` |
| DisplayPort driver | `src/main/io/displayport_msp.c` |
| CRSF tunnel | `src/main/telemetry/crsf.c`, `src/main/rx/crsf.c` |

## Key Entities

- **Betaflight Configurator** — primary MSP client; reference implementation
  of the handshake and save-flow described here.
- **`@betaflight/msp`** (npm) — JS library used by Configurator;
  encodes/decodes V1 + V2 frames, used by other tools as well.
- **ExpressLRS** — main MSP-over-CRSF transport for ELRS users.
- **DJI O3 / Walksnail / HDZero** — three digital VTX systems that consume
  MSP DisplayPort.

## Key Concepts

- [[MSP v2 Frame Format]] — wire format details.
- [[MSP DisplayPort]] — digital OSD over MSP.
- [[MSP API Versioning]] — version handshake + save flow.
- [[MSP over CRSF]] — wireless tunnel.

## Contradictions

None significant. Cross-source agreement on all wire-format facts
(crc8_dvb_s2, $X framing, command IDs). The DeepWiki summary's prose was
treated as a pointer rather than ground truth — exact line numbers were
not load-bearing.

## Open Questions

- **MSP-over-CRSF sequence byte layout**: not in the header file; needs
  source-walk of `crsf.c`.
- **`MSP_DEBUGMSG(253)` buffer behavior**: stated to be a debug string
  buffer; the size and how it's flushed is unclear.
- **MSP_REBOOT MSC variant**: claimed to expose SD as USB MSC — confirm
  payload value and supported boards.
- **MSP_MULTIPLE_MSP framing**: how the response packs multiple sub-replies.
- **Digital VTX font mapping**: bits 0-1 of WRITE_STRING attribute select
  font 0-3, but the mapping to physical font files is vendor-specific —
  no spec found.

## Sources

- [[inav-wiki-msp-v2]] — iNav project, current
- [[betaflight-deepwiki-msp]] — DeepWiki auto-gen, 2025
- [[betaflight-displayport-api]] — Official BF docs
- [[betaflight-msp-protocol-h]] — Primary source code
- [[betaflight-crsf-protocol-h]] — Primary source code
