---
type: meta
title: "Hot Cache"
updated: 2026-05-12T00:00:00
tags: [meta, cache]
---

# Recent Context

## Last Updated
2026-05-12. **Ingest of official BF Getting Started Hardware page**.
Filed 1 expanded source + 7 concept pages + 6 entity pages (sensor
chips). This gives the wiki its first proper "what's on a flight
controller" coverage beyond the MCU.

## Key Recent Facts

- This is a Betaflight research wiki (flight controller firmware for FPV drones)
- Goal: learn BF inside-out — features, internals, configurator workflow, reverse engineering
- Mode E (Research) + Mode B (Repository)
- **Material captured so far**:
  - Ingests: 3 raw articles (FPV autonomy / RPi; 7" long-range build; BF Getting Started Hardware)
  - Autoresearch sessions: 2 (STM32F7x2; STM32 build & RE)

## Key Technical Knowledge

### From This Ingest (BF Getting Started Hardware)

**The FC block inventory** (every BF target maps these blocks to MCU pins):

| Block | Common chip | Bus | Notes |
|-------|-------------|-----|-------|
| MCU | F411/G473/F405/AT32F435/F722/F745/H743 | — | "higher number = more powerful" |
| Voltage regulator | board-specific | — | 3 rails: **3.3 V / 5 V / 9–12 V** |
| OSD chip (analog only) | [[MAX7456]] / AT7456E | SPI | Skipped on digital VTX |
| Gyro + Accel (IMU) | [[MPU6000]] / [[BMI270]] / [[ICM-42688-P]] | SPI | 6-axis combo |
| Barometer | [[BMP280]] / BMP180 / MS5611 / DPS310 | I²C | Hole-in-package; conformal coat kills it |
| Magnetometer | HMC5883L / IST8310 / LIS3MDL | I²C | EMI-sensitive; **not fully integrated in 4.4.1** |
| GPS (external) | [[uBlox GPS Module]] M8 / M10 | UART | M10 auto-config from **BF 4.5.0** |

**IMU sample rates** (matters for looptime defaults):
- MPU6000: 8 kHz
- BMI270: **3.2 kHz** ← lower; clean signal tradeoff
- ICM-42688-P: 8 kHz (32 kHz with FIFO), tier-1 since **BF 4.4.1**

**Voltage rails — overloading kills FCs**:
- 3.3 V → MCU + on-board sensors
- 5 V → RX, analog cam, low-W VTX, LEDs
- 12 V (recommended) → digital VTX (DJI O3, Walksnail, HDZero)

**OSD architectures**:
- Analog video → MAX7456 chip on FC, SPI from MCU, overlay baked into video before VTX
- Digital video → no FC chip, MCU sends MSP DisplayPort over UART, goggles overlay
  - Wins: clean DVR, OSD data export, better fonts

**Barometer hardware gotchas**:
- Package port must stay open (no conformal coat over it)
- Open-cell foam over the chip damps prop-wash pressure spikes

**GPS M10 caveat**:
- BF < 4.5.0: M10 modules need manual u-center config or fix is unreliable
- BF ≥ 4.5.0: auto-config handles M10 → "just works"

### From Autoresearch #2 (STM32 Build & Reverse Engineering)

**Forward path — building firmware**
- Five stages: preprocess → compile (`arm-none-eabi-gcc -c`) → link
  (`arm-none-eabi-ld` + linker script) → object-copy (`objcopy → .bin/.hex`)
  → flash.
- The **linker script** owns the memory layout (`MEMORY` blocks + `SECTIONS`).
  `KEEP(*(.vectors))` forces the [[Vector Table]] to flash offset 0.
- BF target binaries are produced by exactly this pipeline (Cloud Build runs
  it server-side; `make TARGET=STM32F7X2` runs it locally).

**Vector table**: `+0x00` = initial MSP, `+0x04` = Reset_Handler. Thumb mode
function pointer LSB = 1. Moveable via `SCB->VTOR`.

**Reverse path — dumping**: SWD (OpenOCD+ST-Link), USB DFU (`dfu-util`),
UART (`stm32flash`). First 8 bytes of dump = MSP + Reset_Handler.

**Ghidra setup**: `ARM:LE:32:Cortex`, base `0x08000000`, SVD-Loader for
peripheral names.

**Binary patching escalation**: byte flips → veneers (Nexmon) → FPB live
patches (6–8 HW comparators on Cortex-M3/M4/M7).

**STM32 RDP**: L0 readable; L1 falls to ~$200 glitching (ChipWhisperer);
L2 currently unbroken in public research.

### From Autoresearch #1 (STM32F7x2 in Betaflight)
- `STM32F7X2` unified-target covers F722RET6 (512 KB) and F722RGT6 (1 MB).
- F4 / F7 capped at 4 motors for new BF designs; hex/octo requires H7.
- **Cloud Build System** (BF 4.4+) drops unused drivers so 512 KB MCUs fit.
- F7 vs F4: 216 MHz M7 + L1 cache, HW UART inversion, more UARTs.

### From Ingest #1 (MSP Autonomy)
- **MSP Override Mode**: companion computer overrides RC via `MSP_SET_RAW_RC`.
- **MSP v1 frame**: `$M` + dir + size + cmd + payload + XOR.

### From Ingest #2 (7" Long-Range Class)
- 7" cruiser: low KV 1100–1500, big stator, 6S2P Li-Ion, ELRS 915 MHz, GPS Rescue.
- **Li-Ion**: flat discharge curve → use 3.1 V warn / 3.0 V min, NOT BF defaults.
- **SmartAudio**: one-wire VTX control over UART TX.

## Pages in the Vault

### Features
- [[MSP Override Mode]]
- [[SmartAudio]]
- [[CRSF_BAUDRATE]] *(new — source-walk: define in rx/crsf_protocol.h, V3 runtime negotiation)*

### Concepts
- [[Companion Computer]]
- [[Long-Range 7-Inch Class]]
- [[Li-Ion vs LiPo for FPV]]
- [[Betaflight MCU Targets]]
- [[Cloud Build System]]
- [[ARM Cortex-M Firmware Build Process]]
- [[Vector Table]]
- [[Readout Protection (STM32 RDP)]]
- [[Flight Controller Hardware]] *(new)*
- [[OSD (On-Screen Display)]] *(new)*
- [[Inertial Measurement Unit]] *(new)*
- [[Barometer (Altitude Sensing)]] *(new)*
- [[Magnetometer (Compass)]] *(new)*
- [[GPS (Position Sensing)]] *(new)*
- [[FC Voltage Rails]] *(new)*
- [[Finding Defines in Firmware Binaries]] *(new — save from build-env Q&A)*

### Reverse Engineering
- [[MSP Protocol]]
- [[Cortex-M Firmware Dumping]]
- [[Loading Cortex-M Firmware in Ghidra]]
- [[Cortex-M Binary Patching]]
- [[Bin to Hex Conversion and Constant Patching]] *(new — end-to-end recipe: dump → find → patch → reflash, with CRSF_BAUDRATE worked example)*

### Entities
- [[Aocoda F460 Stack]]
- [[SpeedyBee F405 V3 BLS 50A Stack]]
- [[ExpressLRS]]
- [[STM32F722]]
- [[STM32 MCU Family in Betaflight]]
- [[MPU6000]] *(new)*
- [[BMI270]] *(new)*
- [[ICM-42688-P]] *(new)*
- [[MAX7456]] *(new)*
- [[BMP280]] *(new)*
- [[uBlox GPS Module]] *(new)*

### Thesis / Synthesis
- [[Research - STM32F7x2 in Betaflight]]
- [[Research - STM32 Firmware Build and Reverse Engineering]]

## Open Questions / Gaps

### From this ingest
- **GPS Rescue** still has no dedicated page despite being referenced from
  baro / mag / GPS / uBlox — promote from the "next steps" list.
- Magnetometer integration status post-4.4.1: official doc still says
  "not fully integrated in 4.4.1" — what changed in 4.5/4.6?
- MSP DisplayPort details: how exactly does BF push OSD over UART to a
  digital VTX? Frame format, repaint cadence?
- Which BF version brought ICM-42xxx parity with MPU6000 — text says
  "since 4.4.1" — is that the changelog entry?
- Voltage-rail current ratings — is there a manufacturer-spec table
  pattern we could harvest into [[FC Voltage Rails]]?

### Carried over
- Concrete BF binary RE walkthrough end-to-end (autoresearch #2).
- BF bootloader behaviour (PA10 boot pad) vs STM32 ROM DFU.
- Stored config CRC vs hex-edit safety.
- Public Level-2 RDP bypass research status 2026.
- `make/mcu/STM32F7.mk` direct view.
- AT32F435 Configurator target string.
- BF version that introduced MSP Override Mode.
- MSP v2 frame format (16-bit IDs).
- SpeedyBee F405 V3 target name.
- CRSF telemetry packet structure / MSP-over-CRSF.
- SmartAudio v2.1 capability-table quirks.

## Suggested Next Steps

- Create **[[GPS Rescue]]** — now referenced from 4 new pages, biggest
  remaining gap.
- Create **[[Failsafe]]** — referenced from multiple pages.
- Create **[[MSP DisplayPort]]** — the digital-VTX side of OSD.
- Create **[[Betaflight Configurator]]** — Configurator OSD tab is the
  obvious entry point.
- Walk a BF binary through [[Loading Cortex-M Firmware in Ghidra]]
  end-to-end.
- Research STM32H743 entity page (mirror of [[STM32F722]]).
- Research AT32F435 entity page (non-STM alternative).
