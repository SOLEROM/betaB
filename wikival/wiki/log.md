---
type: meta
title: "Operation Log"
created: 2026-05-12
updated: 2026-05-12
tags: [meta, log]
---

# Operation Log

Append-only. Newest entries at the top. Format: `## YYYY-MM-DD — Operation`

---

## 2026-05-12 — save | Bin → Hex Conversion and Constant Patching

- Prompted by: *"write me a page about the process of `.bin → .hex` so
  i can latter find and change some val like the CRSF_BAUDRATE for
  example if it can be done?"*
- Filed: [[Bin to Hex Conversion and Constant Patching]] (protocol,
  reverse) — the practical recipe that stitches together [[Cortex-M Firmware Dumping]] (input), [[Finding Defines in Firmware Binaries]] (theory), [[CRSF_BAUDRATE]] (the value), and [[Cortex-M Binary Patching]] (general theory) into one end-to-end workflow.
- Pages updated: [[Wiki Index]] (count 56 → 57), [[Reverse Engineering Index]].
- Key content filed:
  - Intel HEX format primer (record types: `:00=data`, `:01=EOF`,
    `:04=extended-linear-address`, `:05=start-address`; per-record
    checksum is two's complement of byte sum).
  - The pivotal `objcopy` invocation: `arm-none-eabi-objcopy -I binary
    -O ihex --change-addresses 0x08000000 in.bin out.hex` — forgetting
    `--change-addresses` produces a HEX that says "load at 0x00000000",
    which most flashers will reject or honor literally (bricking the
    board).
  - 7-step workflow for patching CRSF_BAUDRATE = 420000 in a dumped
    .bin: hunt bytes `00 69 06 00`, classify hits (literal pool /
    .data initialiser / collision) via Ghidra xref, replace with
    e.g. `00 10 0E 00` (921600), reconvert to .hex, flash.
  - The **"partial patching breaks the math"** caveat: CRSF_BAUDRATE
    is used in 4 sites including frame-time computation
    (`rx/crsf.c:682`) and the V3 floor check — must patch all sites
    or none.
  - The **`movw/movt` failure mode**: when the constant is inlined
    rather than going through a literal pool, the bytes are spread
    across instruction-encoding bit fields and can't be grepped — need
    real disassembly + reassembly (Ghidra "Modify Instruction" or
    radare2 `wa`).
  - Betaflight-specific bookkeeping: no signature, no `.text` CRC, so
    byte-flips work; but the config region IS CRC'd (separately).
- New gaps captured (2): full BF F405 walkthrough w/ Ghidra
  screenshots, and BF MSP-bootloader integrity-check status.

---

## 2026-05-12 — save | CRSF_BAUDRATE (source-walk)

- Prompted by: *"in the src code of betaflight find the CRSF_BAUDRATE
  and write me in our wiki a page about that param. what it does? what
  we need it for? how change it? where it is stored? how can we change
  that?"*
- Filed: [[CRSF_BAUDRATE]] (feature, area: rc) — first wiki page that
  walks a real BF source identifier end-to-end.
- Pages updated: [[Wiki Index]] (count 55 → 56), [[Features Index]].
- Key findings filed:
  - `CRSF_BAUDRATE` is a `#define` in `rx/crsf_protocol.h:33-37` with
    **two possible values**: 420000 (default, ELRS-compatible) vs
    416666 (`USE_CRSF_OFFICIAL_SPEC`, TBS Rev10 spec — explicitly
    warned against for ELRS STM32 RXs).
  - The 420000 vs 416666 split is a clock-divider story: STM32 UART
    baud generators hit 420000 cleanly from common APB clocks; 416666
    doesn't divide nicely. Both ends running 420000 → no skew.
  - In the built binary the value is **gone as a name** — appears as
    immediate operands (`movw/movt 0x6900/0x6` for 420000 = 0x66900)
    in `crsfRxInit`, `crsfRxUpdateBaudrate`, `getCrsfCachedBaudrate`,
    `getCrsfDesiredSpeed`. This is the worked example for
    [[Finding Defines in Firmware Binaries]].
  - There is **no CLI variable for the baud itself** — the runtime
    escape hatch is `crsf_use_negotiated_baud` (PG-backed in
    `rxConfig_t`, `pg/rx.h:65`), which lets CRSF V3 step up to faster
    rates from the `baudRates[]` table (max **2.47 Mbaud**).
  - Negotiated baud is **persisted in STM32 backup-RAM** via
    `PERSISTENT_OBJECT_SERIALRX_BAUD` (`drivers/persistent.h:40`),
    NOT in the regular config flash region — survives reset but
    depends on FC backup cap for power-cycle survival.
  - `CRSF_BAUDRATE` keeps a structural role even with V3 active: it's
    the **floor** (`getCrsfCachedBaudrate` rejects any cached value
    below it) and the **frame-time reference** (`rx/crsf.c:682`).
- New gaps captured (3): MCU clock-tree tolerances for 416666,
  backup-RAM survival without VBAT/backup-cap, inverted-UART baud
  tolerance.

---

## 2026-05-12 — save | Finding Defines in Firmware Binaries

- Prompted by user question after first successful local BF build: *"what
  are our options regarding recognizing where a `#define` param is set
  in the hex output?"*
- Filed: [[Finding Defines in Firmware Binaries]] (concept,
  domain: build-system) — companion to [[ARM Cortex-M Firmware Build Process]].
- Pages updated: [[Wiki Index]] (count 54 → 55), [[Concepts Index]].
- Key framing captured: a `#define` is preprocessor substitution, not a
  linker symbol — by the time you look at `.hex` the name is gone. What
  you can find depends entirely on how the define was *used* (allocation
  vs immediate vs string vs `#ifdef` gate). Decision table + tooling
  ladder (preprocessor → DWARF → objdump → map → strings → nm →
  diff-build) included.
- Betaflight-specific note recorded: BF Configurator settings are NOT
  defines — they are runtime PG (Parameter Group) structs. To find
  those, grep `.map` for `pgResetFn_` / `pgRegisterFn_` symbols.

---

## 2026-05-12 — correction | F8x2 → F7x2 (typo)

- User clarified the original autoresearch query `stm32f8x2 betaflight
  flight controller` was a typo for `stm32f7x2`.
- Renamed: [[Research - STM32F8x2 (F7x2) in Betaflight]] →
  [[Research - STM32F7x2 in Betaflight]]; dropped the "STM32F8 is a
  non-existent part" section since it was scaffolded only by the typo.
- Deleted: gap page `STM32F8 Series Does Not Exist` (not a real research
  gap — created only because the typo failed to resolve).
- Cross-refs cleaned in [[STM32F722]] and [[STM32 MCU Family in Betaflight]].
- Substantive F7x2 / F722 research is unaffected and retained.

---

## 2026-05-12 — ingest | Betaflight Getting Started Hardware

- Source: `https://betaflight.com/docs/wiki/getting-started/hardware`
- Raw: `.raw/articles/betaflight-getting-started-hardware-2026-05-12.md` (md5 `07af7c4cdd4c2db054648fb9f7fc9782`)
- Summary expanded: [[Betaflight Getting Started Hardware]] (previously a sparse stub from an autoresearch session — now full content)
- Pages created (13):
  - **Concepts (7)**: [[Flight Controller Hardware]], [[OSD (On-Screen Display)]],
    [[Inertial Measurement Unit]], [[Barometer (Altitude Sensing)]],
    [[Magnetometer (Compass)]], [[GPS (Position Sensing)]], [[FC Voltage Rails]]
  - **Entities (6)**: [[MPU6000]], [[BMI270]], [[ICM-42688-P]], [[MAX7456]],
    [[BMP280]], [[uBlox GPS Module]]
- Pages updated: [[Wiki Index]], [[Concepts Index]], [[Entities Index]],
  [[Hot Cache]]
- Key insight: the official BF docs are the source-of-truth for the FC
  block diagram. Every BF target file is a pin-mapping of the seven blocks
  documented here (MCU, regulator, OSD, gyro, baro, mag, GPS).
- Notable specifics filed: MAX7456 / AT7456E clone, MPU6000 8 kHz vs BMI270
  3.2 kHz vs ICM-42688 8 kHz, BMP280 hole-in-package + conformal-coat trap,
  uBlox M10 auto-config since BF 4.5.0, ICM-42xxx parity with MPU6000
  since BF 4.4.1.

---

## 2026-05-12 — autoresearch | STM32 Firmware Build and Reverse Engineering

- Query: `stm32 reverse eng. 1.teach me the concepts and process of building a fw for the device. 2.teach methods to do reverser and changing params of the build binary for cases i dont have the src for my device`
- Rounds: 2 (broad search + gap fill)
- Web searches: 8
- Sources fetched: 6 (1 failed — Doulos FPB article 403; covered via cross-references)
- Pages created (16):
  - **Sources (9)**: [[memfault-linker-scripts]], [[aticleworld-stm32-build]],
    [[wrongbaud-ghidra-stm32-loader]], [[feabhas-ghidra-cortex-m]],
    [[techmaker-stm32-re]], [[giese-defcon26-cortex-m-modify]],
    [[anvil-glitching-stm32-rdp]], [[svd-loader-h2lab]],
    [[cjacker-opensource-toolchain-stm32]]
  - **Concepts (3)**: [[ARM Cortex-M Firmware Build Process]],
    [[Vector Table]], [[Readout Protection (STM32 RDP)]]
  - **Reverse (3)**: [[Cortex-M Firmware Dumping]],
    [[Loading Cortex-M Firmware in Ghidra]],
    [[Cortex-M Binary Patching]]
  - **Thesis (1)**: [[Research - STM32 Firmware Build and Reverse Engineering]]
- Key findings:
  - The Cortex-M vector table is the structural anchor that connects the build
    pipeline (linker script enforces it at flash offset 0) with the RE workflow
    (first 8 bytes give MSP + reset handler, enough to align Ghidra).
  - Dumping path: SWD via OpenOCD/ST-Link, or DFU via dfu-util — both produce
    a flat `.bin` whose first words tell you the chip's SRAM size.
  - Patching escalates: byte flips → veneers in unused flash → FPB live patches.
  - RDP Level 1 falls to ~$200 of glitching hardware (ChipWhisperer / Pi Pico);
    Level 2 is the practical limit but loses SWD/JTAG permanently.

---

## 2026-05-12 — autoresearch | STM32F8x2 (F7x2) in Betaflight

- Query: `stm32f8x2 betaflight flight controller`
- Rounds: 2 (broad search + gap fill)
- Web searches: 8
- Sources fetched: 7 (1 failed — `STM32F7.mk` raw GitHub URL returned 404)
- Pages created (10): [[STM32F722]], [[STM32 MCU Family in Betaflight]],
  [[Betaflight MCU Targets]], [[Cloud Build System]],
  [[STM32F8 Series Does Not Exist]],
  [[Research - STM32F8x2 (F7x2) in Betaflight]],
  [[Oscar Liang FC Processors]], [[Oscar Liang Betaflight 4.4]],
  [[AnyLeaf Quadcopter MCU Comparison]],
  [[Betaflight Manufacturer Design Guidelines]],
  [[Betaflight Issue 197 Supported MCUs]],
  [[Betaflight Configurator Issue 3366 F7X2 Missing]],
  [[DeepWiki Betaflight Config MCU Families]],
  [[Flying Rabbit Creating BF Target]],
  [[Betaflight Getting Started Hardware]]
- Synthesis: [[Research - STM32F8x2 (F7x2) in Betaflight]]
- Key finding: **STM32F8 does not exist** — ST's numbered series jumps
  F7 → H7. The query is a typo for STM32F7x2, the BF target name
  (`STM32F7X2` / `F7X2RE`) covering the F722RET6 (512 KB) and F722RGT6
  (1 MB) parts. The F7 family is "supported but not preferred" — capped
  at 4 motors for new designs, with the H7 family recommended instead.
  The 512 KB F722RE survives the modern feature-set thanks to the
  Cloud Build System introduced in BF 4.4.

---

## 2026-05-12 — ingest | 7-Inch FPV Build Guide (constructive extract)

- Source: `.raw/articles/build-7inch-fpv-drone-2026-05-12.md`
- Summary: [[How to Build a 7-Inch FPV Drone (constructive extract)]]
- Origin framing: military / combat. Extraction mode: **constructive** —
  only the generic FPV drone engineering substrate was captured. Payload,
  weaponization, and target-engagement content from the source was
  intentionally skipped and is not present anywhere in the vault.
- Pages created:
  - [[Long-Range 7-Inch Class]] (concept)
  - [[Li-Ion vs LiPo for FPV]] (concept)
  - [[SpeedyBee F405 V3 BLS 50A Stack]] (entity)
  - [[ExpressLRS]] (entity — promoted from stub)
  - [[SmartAudio]] (feature)
- Pages updated: [[Wiki Index]], [[Features Index]], [[Concepts Index]],
  [[Entities Index]]
- Key insight: the same hardware recipe described in the source — 7"
  airframe, low-KV motors, F4 stack, ELRS 915 MHz, analog VTX, Li-Ion
  6S2P — is the canonical **long-range cruiser** configuration used in
  search-and-rescue, wildfire spotting, agricultural scouting, infra
  inspection, and long-range cinematography. The engineering is dual-use
  by physics, not by intent.

---

## 2026-05-12 — ingest | FPV Autonomous Operation with Betaflight and Raspberry Pi

- Source: `.raw/articles/fpv-autonomous-operation-betaflight-rpi-2026-05-12.md`
- Summary: [[FPV Autonomous Operation with Betaflight and Raspberry Pi]]
- Pages created: [[MSP Override Mode]], [[MSP Protocol]], [[Companion Computer]], [[Aocoda F460 Stack]]
- Pages updated: [[Features Index]], [[Concepts Index]], [[Reverse Engineering Index]], [[Entities Index]], [[Wiki Index]]
- Key insight: Betaflight's MSP OVERRIDE mode + `msp_override_channels_mask` is the bridge that lets a Raspberry Pi companion computer fly a drone autonomously while BF's PID loop and safety systems stay active.

---

## 2026-05-12 — SCAFFOLD

- Created vault structure: features/, concepts/, architecture/, configurator/, reverse/, entities/, thesis/, gaps/
- Created CLAUDE.md, wiki/index.md, wiki/log.md, wiki/hot.md, wiki/overview.md
- Created _index.md for all 8 sections with stub page lists
- Created 5 templates in _templates/
- Created CSS snippet at .obsidian/snippets/vault-colors.css
- Initialized git
- Mode: E (Research) + B (Repository)
- Purpose: Betaflight flight control software — features, internals, configurator, reverse engineering
