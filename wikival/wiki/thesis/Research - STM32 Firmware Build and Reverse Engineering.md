---
type: synthesis
title: "Research: STM32 Firmware Build and Reverse Engineering"
created: 2026-05-12
updated: 2026-05-12
tags:
  - research
  - stm32
  - reverse-engineering
  - build-system
status: developing
related:
  - "[[ARM Cortex-M Firmware Build Process]]"
  - "[[Vector Table]]"
  - "[[Readout Protection (STM32 RDP)]]"
  - "[[Cortex-M Firmware Dumping]]"
  - "[[Loading Cortex-M Firmware in Ghidra]]"
  - "[[Cortex-M Binary Patching]]"
  - "[[STM32F722]]"
  - "[[STM32 MCU Family in Betaflight]]"
  - "[[Betaflight MCU Targets]]"
  - "[[Cloud Build System]]"
sources:
  - "[[memfault-linker-scripts]]"
  - "[[aticleworld-stm32-build]]"
  - "[[wrongbaud-ghidra-stm32-loader]]"
  - "[[feabhas-ghidra-cortex-m]]"
  - "[[techmaker-stm32-re]]"
  - "[[giese-defcon26-cortex-m-modify]]"
  - "[[anvil-glitching-stm32-rdp]]"
  - "[[svd-loader-h2lab]]"
  - "[[cjacker-opensource-toolchain-stm32]]"
---

# Research: STM32 Firmware Build and Reverse Engineering

## Overview
Two halves of the same loop. Building firmware is the **forward** direction: C source → object files → linked ELF → flat `.bin` → flash. Reverse engineering and patching is the **inverse**: flat `.bin` → recovered disassembly → modified bytes → reflash. The artefact that connects them is the [[Vector Table]] — a fixed-format anchor every Cortex-M binary carries at flash offset 0.

This synthesis answers the two-part research brief:
1. *Build a firmware for the device* — covered by [[ARM Cortex-M Firmware Build Process]].
2. *Reverse a built binary and modify its parameters without source* — covered by [[Cortex-M Firmware Dumping]] → [[Loading Cortex-M Firmware in Ghidra]] → [[Cortex-M Binary Patching]], with [[Readout Protection (STM32 RDP)]] as the obstacle on locked targets.

## Key Findings

### Build path

- **Five stages** turn source into a flash image: preprocess → compile → link → object-copy → flash. The compiler is `arm-none-eabi-gcc`; the linker is `arm-none-eabi-ld`; the format converter is `arm-none-eabi-objcopy` (Source: [[aticleworld-stm32-build]], [[cjacker-opensource-toolchain-stm32]]).
- **The linker script** is the contract between hardware and binary. It declares MEMORY regions (flash at `0x08000000`, SRAM at `0x20000000`), forces the vector table to flash offset 0 with `KEEP(*(.vectors))`, and emits the LMA/VMA pairs the reset handler needs to copy `.data` into RAM (Source: [[memfault-linker-scripts]]).
- **The reset handler** runs before `main()`: it copies `.data` from LMA to VMA, zeroes `.bss`, configures the clock tree, then calls `main()`.
- **Betaflight runs this pipeline** for every TARGET. The Cloud Build System ([[Cloud Build System]]) does it server-side; Developer Mode and `make TARGET=STM32F7X2` do it locally with the same toolchain.

### Reverse path

- **Dump first** ([[Cortex-M Firmware Dumping]]). Four entrypoints exist: OpenOCD+ST-Link (SWD), `st-flash` (SWD), `dfu-util` (USB DFU via system bootloader), `stm32flash` (UART). Output is a flat `.bin` whose first 8 bytes are MSP + Reset_Handler (Source: [[techmaker-stm32-re]]).
- **Anchor on the vector table** ([[Vector Table]]). Two 32-bit words are enough to identify SRAM size, entry point, and chip family.
- **Disassemble** ([[Loading Cortex-M Firmware in Ghidra]]). Pick `ARM:LE:32:Cortex`, set base `0x08000000`, run [[SVD-Loader]] to label peripherals, navigate to `Reset_Handler`, hit auto-analyse with Aggressive Instruction Finder. radare2 with `-a arm -b 16 -m 0x08000000` does the same job from the CLI (Source: [[wrongbaud-ghidra-stm32-loader]], [[feabhas-ghidra-cortex-m]]).
- **Patch** ([[Cortex-M Binary Patching]]). Three levels of escalation:
  1. **Byte flips** for constants and conditional branches.
  2. **Veneers** in unused flash for new behaviour, with a `BL` redirected from the call site.
  3. **Flash Patch & Breakpoint (FPB)** for live runtime patches without modifying flash.
  Frameworks: Nexmon for C-level Cortex-M patching, `arm-none-eabi-as` for hand-written assembly veneers.

### Locked targets
- **RDP Level 0** — default, fully readable.
- **RDP Level 1** — readable only with hardware attacks: voltage fault injection (Anvil, CTXz `stm32f1-picopwner`, Joe Grand 2022) or the FPB-exception bug on STM32F1 (Obermaier/Schink/Moczek).
- **RDP Level 2** — currently unbroken in public research; the cost is permanent loss of SWD/JTAG (Source: [[anvil-glitching-stm32-rdp]]).

## Key Entities
- [[Ghidra]] — primary RE platform (NSA, open source).
- [[SVD-Loader]] — peripheral labeling.
- **radare2 / Cutter** — CLI RE alternative.
- **Nexmon** — C-based binary patching for Cortex-M.
- **OpenOCD / ST-Link / dfu-util / st-flash** — programming and dumping tools.
- [[arm-none-eabi-gcc]] — the toolchain (also used by Betaflight upstream).

## Key Concepts
- [[ARM Cortex-M Firmware Build Process]] — the forward pipeline.
- [[Vector Table]] — anchor for both build and RE.
- [[Readout Protection (STM32 RDP)]] — the lock.
- [[Voltage Fault Injection]] — the lock-picking technique.

## Bridge to Betaflight

Why this matters for the Betaflight wiki:

1. **Building**: BF's Makefile is exactly the pipeline in [[ARM Cortex-M Firmware Build Process]]. Per-TARGET `.ld` linker scripts live under `src/main/target/<TARGET>/`. Cloud Build invokes the same chain server-side. Reading a BF `.hex` is a special case of reading any STM32 firmware.
2. **Reversing**: if a closed-source competitor (Helio, INAV-derivative, vendor BF fork) ships a binary without source, the workflow in [[Loading Cortex-M Firmware in Ghidra]] applies unchanged. Anchor on the vector table, load CMSIS SVD for the right STM32 family ([[STM32 MCU Family in Betaflight]]), and start with the `cli` string xrefs.
3. **Patching**: useful for recovering bricked boards (hex-edit a known-good `.bin`), for unlocking features stripped by Cloud Build (the legitimate path is to re-build, but FPB can re-add them at runtime), or for studying retail vendor customisations.

## Contradictions
- None substantive among the sources fetched. Build-process accounts converge; RE workflows differ only in tool choice (Ghidra vs radare2 vs IDA).

## Open Questions
- Concrete BF-specific worked example: dump a Betaflight `.bin`, locate the `motor_pwm_rate` setting in the EEPROM blob, patch it, reflash.
- Behaviour of the Betaflight bootloader (PA10 boot pad pull) vs the STM32 ROM DFU bootloader — which takes priority, and how to force one or the other.
- Does Betaflight's stored config carry any CRC that would block a hex-edit?
- Status of public Level-2 RDP bypass research as of 2026 (the 2017 Wojtczuk paper and successors).
- Whether STM32H7 secure boot (with key authority) is in any production BF target — would change the patching story significantly.

## Sources
- [[memfault-linker-scripts]] — Tyler Hoffman, Memfault — linker scripts deep dive
- [[aticleworld-stm32-build]] — Amlendra Kumar — stage-by-stage build pipeline
- [[wrongbaud-ghidra-stm32-loader]] — wrongbaud — custom Ghidra STM32 loader
- [[feabhas-ghidra-cortex-m]] — Feabhas — raw binary disassembly with Ghidra
- [[techmaker-stm32-re]] — TechMaker / Medium — OpenOCD + radare2 RE workflow
- [[giese-defcon26-cortex-m-modify]] — Dennis Giese, DEFCON 26 — Cortex-M firmware modification
- [[anvil-glitching-stm32-rdp]] — Anvil Secure — voltage fault injection on STM32 RDP
- [[svd-loader-h2lab]] — leveldown-security — Ghidra peripheral loader plugin
- [[cjacker-opensource-toolchain-stm32]] — cjacker — complete OSS STM32 toolchain guide
