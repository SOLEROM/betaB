---
type: meta
title: "Hot Cache"
updated: 2026-05-14T00:00:00
tags: [meta, cache]
---

# Recent Context

## Last Updated
2026-05-14. **Autoresearch: MSP protocol controlling Betaflight firmware.**
10 new pages filed (4 concepts, 5 sources, 1 synthesis) + the existing
[[MSP Protocol]] page promoted from `developing` → `documented` with gaps
resolved. Wiki now has end-to-end coverage of how host tooling
(Configurator, companion computers, goggles) talks to the FC.

## Key Recent Facts

- Betaflight research wiki (flight controller firmware for FPV drones)
- Goal: learn BF inside-out — features, internals, configurator, RE
- Mode E (Research) + Mode B (Repository)
- **Material captured so far**:
  - Ingests: 3 raw articles
  - Autoresearch sessions: 3 (STM32F7x2; STM32 build & RE; **MSP protocol**)

## Key Technical Knowledge

### From This Autoresearch (MSP Protocol)

**MSP is the single wire surface** for everything except realtime RC:
configuration, telemetry, persistence, reboot, OSD on digital VTX, even
wireless config tunneling through ExpressLRS.

**Frame formats** ([[MSP v2 Frame Format]]):
- **V1**: `$M` + dir + size(u8) + cmd(u8) + payload + XOR-sum. Max 255 bytes.
- **V2**: `$X` + dir + flags(u8) + func(u16-LE) + size(u16-LE) + payload + crc8_dvb_s2. Max 65535 bytes.
- **Tunneling**: V2 wrapped in V1 frame using cmd=255 (`MSP_V2_FRAME`).
- CRC: `crc8_dvb_s2`, polynomial **0xD5**, init 0, covers flags onward.

**MSP v2 ID partitioning** ([[betaflight-msp-protocol-h]]):
- `0x0000–0x00FF` → mirrors V1 commands
- `0x1000+` → `MSP2_COMMON_*`
- `0x1F00+` → `MSP2_SENSOR_*`
- `0x3000+` → `MSP2_BETAFLIGHT_*` (e.g. `MSP2_BETAFLIGHT_BIND = 0x3000`)

**Configurator handshake** ([[MSP API Versioning]]):
1. `MSP_API_VERSION(1)` → [proto_ver, major, minor] — drives feature gating
2. `MSP_FC_VARIANT(2)` → 4-byte ASCII (`BTFL` for Betaflight)
3. `MSP_FC_VERSION(3)` → [major, minor, patch]
4. `MSP_BUILD_INFO(5)` → date + time + git short hash
5. `MSP_BOARD_INFO(4)` → board ID + hw revision

Current Betaflight API: **1.47** (BF 4.6); 1.46 was BF 4.5.

**Save-settings flow**: `MSP_SET_*` × N → `MSP_EEPROM_WRITE(250)` →
`MSP_REBOOT(68)`. Both write/reboot abort if armed. EEPROM_WRITE calls
`writeEEPROM()` then `readEEPROM()` and re-inits VTX table.

**MSP_REBOOT variants**: 0 = firmware, 1 = bootloader (DFU), 2 = MSC, 3 = MSC with UTC.

**MSP DisplayPort** ([[MSP DisplayPort]]) — `MSP_DISPLAYPORT(182)`:
| Sub | Name | Payload |
|-----|------|---------|
| 0 | HEARTBEAT | none |
| 1 | RELEASE | none |
| 2 | CLEAR_SCREEN | none |
| 3 | WRITE_STRING | row, col, attr, ASCIIZ ≤30 |
| 4 | DRAW_SCREEN | none |
| 5 | OPTIONS | resolution (not used by BF) |
| 6 | SYS | row, col, system_element_id |

Attribute byte: bit 6 = blink, bits 0-1 = font number (4 × 256 glyphs).
Repaint loop: `CLEAR_SCREEN → WRITE_STRING* → DRAW_SCREEN` at ~30 Hz.

**MSP over CRSF** ([[MSP over CRSF]]):
- `CRSF_FRAMETYPE_MSP_REQ` = `0x7A` (request)
- `CRSF_FRAMETYPE_MSP_RESP` = `0x7B` (58-byte chunks, FC→host)
- `CRSF_FRAMETYPE_MSP_WRITE` = `0x7C` (8-byte chunks, host→FC — OpenTX limit)
- Why writes are slow: 8-byte outbound chunks force many CRSF frames per MSP write.

**Implementation anchors**:
- Frame parser: `src/main/msp/msp_serial.c` → `mspSerialProcessReceivedData()`
- V1 IDs: `src/main/msp/msp_protocol.h`
- V2 BF IDs: `src/main/msp/msp_protocol_v2_betaflight.h`
- V2 common: `src/main/msp/msp_protocol_v2_common.h`
- Dispatcher: `src/main/msp/msp.c` → `mspCommonProcessOutCommand()` + `mspFcProcessOutCommand()`
- DisplayPort driver: `src/main/io/displayport_msp.c`
- CRSF tunnel: `src/main/telemetry/crsf.c` + `src/main/rx/crsf.c`

**Configurator JS lib**: `@betaflight/msp` (npm) — encodes/decodes V1+V2.

### From Autoresearch #2 (STM32 Build & RE)

- Forward pipeline: preprocess → compile → link (`KEEP(*(.vectors))` puts
  vector table at flash 0x08000000) → objcopy → flash.
- Vector table: `+0x00` MSP, `+0x04` Reset_Handler. Thumb bit (LSB=1).
- Dump: SWD (OpenOCD), USB DFU (`dfu-util`), UART (`stm32flash`).
- Ghidra: `ARM:LE:32:Cortex`, base `0x08000000`, SVD-Loader.
- RDP: L0 readable; L1 ~$200 glitch; L2 currently unbroken.

### From Autoresearch #1 (STM32F7x2 in Betaflight)
- `STM32F7X2` covers F722RET6 (512K) and F722RGT6 (1M).
- F4/F7 capped at 4 motors; hex/octo → H7.
- Cloud Build System (4.4+) drops unused drivers for 512K MCUs.
- F7 vs F4: 216 MHz M7 + L1 cache, HW UART inversion, more UARTs.

### From Ingest #3 (BF Getting Started Hardware)
- FC block inventory: MCU + voltage regulator + OSD chip (MAX7456 analog
  only) + IMU + baro + mag + GPS.
- Voltage rails: 3.3 / 5 / 12 V — overloading 5V or 12V kills FCs.
- BMI270 sampled at 3.2 kHz vs MPU6000 8 kHz.
- BF 4.5+ auto-configures uBlox M10 GPS.

## Pages in the Vault

### Features
- [[MSP Override Mode]]
- [[SmartAudio]]
- [[CRSF_BAUDRATE]]

### Concepts
- [[Companion Computer]]
- [[Long-Range 7-Inch Class]]
- [[Li-Ion vs LiPo for FPV]]
- [[Betaflight MCU Targets]]
- [[Cloud Build System]]
- [[ARM Cortex-M Firmware Build Process]]
- [[Vector Table]]
- [[Readout Protection (STM32 RDP)]]
- [[Flight Controller Hardware]]
- [[OSD (On-Screen Display)]]
- [[Inertial Measurement Unit]]
- [[Barometer (Altitude Sensing)]]
- [[Magnetometer (Compass)]]
- [[GPS (Position Sensing)]]
- [[FC Voltage Rails]]
- [[Finding Defines in Firmware Binaries]]
- **[[MSP v2 Frame Format]]** *(new)*
- **[[MSP DisplayPort]]** *(new)*
- **[[MSP API Versioning]]** *(new)*
- **[[MSP over CRSF]]** *(new)*

### Reverse Engineering
- [[MSP Protocol]] *(upgraded → documented; gaps resolved)*
- [[Cortex-M Firmware Dumping]]
- [[Loading Cortex-M Firmware in Ghidra]]
- [[Cortex-M Binary Patching]]
- [[Bin to Hex Conversion and Constant Patching]]

### Entities
- [[Aocoda F460 Stack]]
- [[SpeedyBee F405 V3 BLS 50A Stack]]
- [[ExpressLRS]]
- [[STM32F722]]
- [[STM32 MCU Family in Betaflight]]
- [[MPU6000]] / [[BMI270]] / [[ICM-42688-P]]
- [[MAX7456]] / [[BMP280]] / [[uBlox GPS Module]]

### Thesis / Synthesis
- [[Research - STM32F7x2 in Betaflight]]
- [[Research - STM32 Firmware Build and Reverse Engineering]]
- **[[Research - MSP Protocol Controlling Betaflight Firmware]]** *(new)*

## Open Questions / Gaps

### From this autoresearch
- MSP-over-CRSF sequence/status byte layout (needs `crsf.c` source-walk).
- `MSP_DEBUGMSG(253)` buffer size + flush semantics.
- `MSP_MULTIPLE_MSP(230)` response framing — how sub-replies pack.
- Digital VTX font mapping (bits 0-1 = font 0-3 — vendor-specific physical mapping).
- MSP_REBOOT MSC variant — payload value and supported boards.

### Carried over from prior sessions
- GPS Rescue page (referenced 4 times, still missing).
- Failsafe page.
- BF version that introduced MSP Override Mode.
- Magnetometer integration status post-4.4.1.
- SmartAudio v2.1 capability-table quirks.
- AT32F435 Configurator target string.
- STM32H743 entity page.
- Public Level-2 RDP bypass research status 2026.

## Suggested Next Steps

- Create **[[Betaflight Configurator]]** — the canonical MSP client, now
  well-anchored by the new MSP pages.
- Source-walk `src/main/telemetry/crsf.c` to close the MSP-over-CRSF
  sequence-byte gap.
- Create **[[GPS Rescue]]** and **[[Failsafe]]** — long-standing gaps.
- Capture a live MSP trace from `betaflight_SITL.elf` (sitl/) and annotate
  the handshake bytes against [[MSP API Versioning]].
- Walk a BF binary through [[Loading Cortex-M Firmware in Ghidra]] end-to-end.
