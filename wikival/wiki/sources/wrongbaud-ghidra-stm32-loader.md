---
type: source
source_type: blog
title: "Writing a GHIDRA Loader: STM32 Edition"
author: "wrongbaud"
url: https://wrongbaud.github.io/posts/writing-a-ghidra-loader/
confidence: high
fetched: 2026-05-12
tags: [source, ghidra, reverse-engineering, stm32]
---

# wrongbaud — Ghidra STM32 Loader

## What it covers
Builds a custom Ghidra loader plugin for STM32 raw flash dumps. Walks through the manual import steps anyone can perform without the plugin, then automates them.

## Key claims
- **Import as flat binary** — raw STM32 dumps have no ELF headers; manual format choice required.
- **Language**: `ARM Cortex (Little Endian)`.
- **Base address**: `0x08000000` (STM32 flash start).
- **Three memory blocks** must be defined: flash (`0x08000000`), SRAM (`0x20000000`), MMIO peripherals (varies; e.g. TIM2 at `0x40000000`).
- **Vector table parsing** — first 32-bit word = initial Main Stack Pointer; second 32-bit word = Reset handler address. The loader iterates subsequent interrupt vectors and creates labeled entry points.
- **Auto-loader vs manual**: the plugin reads vectors and creates the memory map; without it, you do the same work via the Ghidra GUI.

## Why it matters
Concrete recipe for opening any STM32 dump (including a Betaflight `.bin`) and getting Ghidra to disassemble it correctly. Primary reference for [[Loading Cortex-M Firmware in Ghidra]].

## Confidence
**high** — author publishes hardware-RE tutorials with reproducible steps and working code on GitHub.
