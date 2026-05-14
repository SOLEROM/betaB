---
type: source
source_type: tutorial
title: "STM32 Build Process Explained — A Practical Guide for Firmware Engineers"
author: "Amlendra Kumar (aticleworld.com)"
url: https://aticleworld.com/stm32-build-process/
confidence: medium
fetched: 2026-05-12
tags: [source, build-system, stm32]
---

# Aticleworld — STM32 Build Process

## What it covers
End-to-end pipeline: preprocessing → compilation → assembly → linking → objcopy → flash. Uses IAR's `iccarm` / `ielfdumparm` in the example but the stages are identical for the GNU `arm-none-eabi-*` chain.

## Key claims
- **Stage sequence**: Source → Compiler → `.o` (relocatable ELF) → Linker → Final ELF → HEX/BIN.
- **Object files** carry machine instructions without final addresses, plus symbol tables and relocation records.
- **The linker** resolves cross-file symbols and assigns final memory addresses using the linker script.
- **Object-copy stage** converts the linked ELF into the flat `.hex` or `.bin` that the programmer uploads.

## Why it matters
Companion to [[memfault-linker-scripts]] — names the toolchain stages explicitly so the reader can map them onto the GNU equivalents (`arm-none-eabi-gcc -c`, `arm-none-eabi-ld`, `arm-none-eabi-objcopy`, `arm-none-eabi-size`).

## Confidence
**medium** — practitioner blog, IAR-flavoured but factually correct for the generic pipeline.
