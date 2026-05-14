---
type: meta
title: "Wiki Index"
created: 2026-05-12
updated: 2026-05-12
tags: [meta, index]
---

<!-- updated 2026-05-14 — autoresearch: MSP protocol controlling Betaflight FW -->


# Betaflight Wiki — Master Index

**Purpose:** Deep-dive research on Betaflight flight control software — features, internals, configurator, and reverse engineering.

**Mode:** E (Research) + B (Repository)

---

## Sections

| Section | Description | Index |
|---------|-------------|-------|
| Features | Major BF features: PID, filtering, OSD, GPS, modes | [[Features Index]] |
| Concepts | Flight control theory, signal chain, PID math | [[Concepts Index]] |
| Architecture | Codebase modules, scheduler, HAL, FC targets | [[Architecture Index]] |
| Configurator | BF Configurator tabs, CLI, settings | [[Configurator Index]] |
| Reverse | MSP protocol, binary formats, RE notes | [[Reverse Engineering Index]] |
| Entities | FC hardware, ESCs, developers, vendors, tools | [[Entities Index]] |
| Thesis | Evolving synthesis: how BF actually works | [[Thesis Index]] |
| Gaps | Open questions, undocumented behavior | [[Gaps Index]] |

---

## All Pages

*Updated on every ingest. Format: [[Page]] — type — status*

### Features

- [[MSP Override Mode]] — feature — developing
- [[SmartAudio]] — feature — developing
- [[CRSF_BAUDRATE]] — feature — developing

### Concepts

- [[Companion Computer]] — concept — developing
- [[Long-Range 7-Inch Class]] — concept — developing
- [[Li-Ion vs LiPo for FPV]] — concept — developing
- [[Betaflight MCU Targets]] — concept — developing
- [[Cloud Build System]] — concept — developing
- [[ARM Cortex-M Firmware Build Process]] — concept — developing
- [[Finding Defines in Firmware Binaries]] — concept — developing
- [[Vector Table]] — concept — developing
- [[Readout Protection (STM32 RDP)]] — concept — developing
- [[Flight Controller Hardware]] — concept — developing
- [[OSD (On-Screen Display)]] — concept — developing
- [[Inertial Measurement Unit]] — concept — developing
- [[Barometer (Altitude Sensing)]] — concept — developing
- [[Magnetometer (Compass)]] — concept — developing
- [[GPS (Position Sensing)]] — concept — developing
- [[FC Voltage Rails]] — concept — developing
- [[MSP v2 Frame Format]] — concept — documented
- [[MSP DisplayPort]] — concept — documented
- [[MSP API Versioning]] — concept — documented
- [[MSP over CRSF]] — concept — documented

### Architecture

**Master source-tree walkthrough** ([[srclatest/_index|srclatest/_index]] — read top-down):

- [[srclatest/01-overview]] — architecture — stable — 5-layer model, gyro-as-heartbeat, PLATFORM/TARGET/CONFIG triaxis
- [[srclatest/02-directory-layout]] — architecture — stable — `src/main/`, `src/platform/`, `lib/main/` tree
- [[srclatest/03-build-system]] — architecture — stable — Makefile, `mk/*.mk`, toolchain, outputs
- [[srclatest/04-boot-and-scheduler]] — architecture — stable — `main()`, 3-phase init, cooperative scheduler
- [[srclatest/05-flight-core-loop]] — architecture — stable — `taskMainPidLoop` subtask chain, modes, arming
- [[srclatest/06-flight-modules]] — architecture — stable — `flight/` + `fc/` file inventory
- [[srclatest/07-hal-and-drivers]] — architecture — stable — HAL pattern, `drivers/` + `sensors/` + platform split
- [[srclatest/08-io-subsystems]] — architecture — stable — `io/` inventory (serial, VTX, GPS, LED, beeper, …)
- [[srclatest/09-msp-cli-cms]] — architecture — stable — three config interfaces converging on PGs
- [[srclatest/10-osd-blackbox-telemetry]] — architecture — stable — OSD elements, logger, downlink telemetry
- [[srclatest/11-rx-subsystem]] — architecture — stable — serial RX + SPI RX (ExpressLRS, CC2500, …)
- [[srclatest/12-config-and-pg]] — architecture — stable — Parameter Groups, EEPROM, versioning
- [[srclatest/13-modification-guide]] — how-to — stable — cookbook: where to edit to change X

### Configurator
*(none yet)*

### Reverse Engineering

- [[MSP Protocol]] — protocol — documented
- [[Cortex-M Firmware Dumping]] — protocol — documented
- [[Loading Cortex-M Firmware in Ghidra]] — protocol — documented
- [[Cortex-M Binary Patching]] — protocol — documented
- [[Bin to Hex Conversion and Constant Patching]] — protocol — documented
- [[MSP Commands Reference]] — protocol — stub
- [[CRSF Protocol]] — protocol — stub
- [[Blackbox Format]] — protocol — stub
- [[Bootloader]] — protocol — stub
- [[EEPROM Layout]] — protocol — stub
- [[CLI Internals]] — protocol — stub
- [[OSD Font Format]] — protocol — stub
- [[Build System RE]] — protocol — stub

### Entities

- [[Aocoda F460 Stack]] — entity — active
- [[SpeedyBee F405 V3 BLS 50A Stack]] — entity — active
- [[ExpressLRS]] — entity — active
- [[STM32F722]] — entity — active
- [[STM32 MCU Family in Betaflight]] — entity — active
- [[MPU6000]] — entity — active
- [[BMI270]] — entity — active
- [[ICM-42688-P]] — entity — active
- [[MAX7456]] — entity — active
- [[BMP280]] — entity — active
- [[uBlox GPS Module]] — entity — active

### Sources

- [[FPV Autonomous Operation with Betaflight and Raspberry Pi]] — source — ingested
- [[How to Build a 7-Inch FPV Drone (constructive extract)]] — source — ingested
- [[Oscar Liang FC Processors]] — source — ingested
- [[Oscar Liang Betaflight 4.4]] — source — ingested
- [[AnyLeaf Quadcopter MCU Comparison]] — source — ingested
- [[Betaflight Manufacturer Design Guidelines]] — source — ingested
- [[Betaflight Issue 197 Supported MCUs]] — source — ingested
- [[Betaflight Configurator Issue 3366 F7X2 Missing]] — source — ingested
- [[DeepWiki Betaflight Config MCU Families]] — source — ingested
- [[Flying Rabbit Creating BF Target]] — source — ingested
- [[Betaflight Getting Started Hardware]] — source — ingested
- [[memfault-linker-scripts]] — source — ingested
- [[aticleworld-stm32-build]] — source — ingested
- [[wrongbaud-ghidra-stm32-loader]] — source — ingested
- [[feabhas-ghidra-cortex-m]] — source — ingested
- [[techmaker-stm32-re]] — source — ingested
- [[giese-defcon26-cortex-m-modify]] — source — ingested
- [[anvil-glitching-stm32-rdp]] — source — ingested
- [[svd-loader-h2lab]] — source — ingested
- [[cjacker-opensource-toolchain-stm32]] — source — ingested
- [[inav-wiki-msp-v2]] — source — ingested
- [[betaflight-deepwiki-msp]] — source — ingested
- [[betaflight-displayport-api]] — source — ingested
- [[betaflight-msp-protocol-h]] — source — ingested
- [[betaflight-crsf-protocol-h]] — source — ingested

### Thesis

- [[Research - STM32F7x2 in Betaflight]] — synthesis — developing
- [[Research - STM32 Firmware Build and Reverse Engineering]] — synthesis — developing
- [[Research - MSP Protocol Controlling Betaflight Firmware]] — synthesis — developing

### Gaps
*(none yet)*

---

## Stats

- Total pages: 75
- Last updated: 2026-05-14
- Sources ingested: 3 (raw articles) + 23 (autoresearch web sources) = 26
