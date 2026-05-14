---
type: protocol
title: "Loading Cortex-M Firmware in Ghidra"
status: documented
direction: host-only
transport: filesystem
created: 2026-05-12
updated: 2026-05-12
tags: [protocol, reverse, ghidra, cortex-m]
---

# Loading Cortex-M Firmware in Ghidra

## Overview
A raw `.bin` dump from a Cortex-M device has no headers, no symbols, and no idea where it lives. Ghidra needs five facts to disassemble it correctly:

1. **Architecture** — `ARM:LE:32:Cortex` (or `ARM v7 little endian` for M4).
2. **Base address** — `0x08000000` for STM32 (other vendors differ).
3. **Memory blocks** — flash, SRAM, MMIO.
4. **Entry point** — derived from vector table.
5. **Thumb mode** — every function on Cortex-M is Thumb-2.

(Sources: [[wrongbaud-ghidra-stm32-loader]], [[feabhas-ghidra-cortex-m]])

## Step-by-Step

### 1. Import as raw binary
File → Import → select `firmware.bin`. In the *Format* dropdown choose **Raw Binary**.

### 2. Pick the processor
Language → search "Cortex". Pick `ARM:LE:32:Cortex` (or for M0, `ARM:LE:32:v6-M`).

### 3. Set base address
Options → Block Name `flash`, Block Start `0x08000000`. Hit OK.

### 4. Add SRAM and MMIO blocks
After import, Window → Memory Map → Add:

| Name | Start | Length | R/W/X | Type |
|------|-------|--------|-------|------|
| flash | 0x08000000 | 0x00080000 | R-X | Initialised (file) |
| sram | 0x20000000 | 0x00040000 | RWX | Uninit |
| peripheral | 0x40000000 | 0x10000000 | RW | Uninit |
| ppb | 0xE0000000 | 0x00100000 | RW | Uninit |

(STM32F722 sizes shown — adjust per chip.)

### 5. Disassemble the reset vector
Navigate to `0x08000004`. The four bytes are a function pointer with LSB set. Clear the LSB mentally — the real entry point is the address with bit 0 = 0. Right-click → Disassemble. Press `F` to define a function. Press `L` to label it `Reset_Handler`.

### 6. Re-run Auto-Analysis
Analysis → Auto Analyze → enable **ARM Aggressive Instruction Finder**. This finds Thumb functions Ghidra missed (Source: [[feabhas-ghidra-cortex-m]]).

### 7. Apply SVD-Loader
Window → Script Manager → run `SVD-Loader.py`. Point it at the SVD file from `arm.com` or the STM32Cube package for your exact MCU. Every peripheral register address turns into a labeled struct field (Source: [[svd-loader-h2lab]]).

### 8. Anchor with strings
Window → Defined Strings. Filter for distinctive text: error messages, version strings, MSP command names like `"BTFL"`. Each string's xrefs lead you to the functions that touched it — the fastest way to find UI/feature code in an unknown binary.

## Common Pitfalls

- **Flash aliasing** — STM32 also presents flash at `0x00000000` on reset. If you import at `0x00000000` instead of `0x08000000`, all the absolute pointers in the vector table will be wrong by one bank. Always use `0x08000000` (Source: [[feabhas-ghidra-cortex-m]]).
- **Forgotten Thumb bit** — function pointers have LSB = 1. The disassembled function lives at the *cleared* address.
- **Wrong SRAM size** — picking too-small SRAM block in step 4 makes MSP look invalid; double-check the chip's RM.

## Equivalent in radare2
```
r2 -a arm -b 16 -m 0x08000000 -w firmware.bin
> aaa                # auto-analyse
> pdf @ 0x080001ad   # print disassembly at reset
```
(Source: [[techmaker-stm32-re]])

## Sources
- [[wrongbaud-ghidra-stm32-loader]]
- [[feabhas-ghidra-cortex-m]]
- [[svd-loader-h2lab]]
- [[techmaker-stm32-re]]
