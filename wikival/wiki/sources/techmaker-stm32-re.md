---
type: source
source_type: blog
title: "Reverse engineering STM32 firmware"
author: "Alexander Olenyev (TechMaker / Medium)"
url: https://medium.com/techmaker/reverse-engineering-stm32-firmware-578d53e79b3
confidence: medium
fetched: 2026-05-12
tags: [source, reverse-engineering, radare2, openocd]
---

# TechMaker — Reverse Engineering STM32 Firmware

## Key claims
- **Dump with OpenOCD + ST-Link**:
  ```
  openocd -f interface/stlink-v2.cfg -f target/stm32f1x.cfg \
          -c "init" -c "reset init" \
          -c "flash read_bank 0 firmware.bin 0 0x8000" \
          -c "exit"
  ```
- **Triage strings**: `strings firmware.bin` reveals readable text and signals presence/absence of encryption.
- **Entropy visualisation**: `binvis.io` shows code vs data vs compressed regions.
- **radare2 invocation for Cortex-M**:
  ```
  r2 -a arm -b 16 -m 0x08000000 -w firmware.bin
  ```
  `-b 16` for Thumb, `-m` sets base.
- **Reset handler** lives at `0x08000004` (the second 32-bit word).
- Use `aaa` (auto-analysis) and `pdf` (print disassembled function) inside r2.
- **Patch strategy**: replace conditional branches (`cbnz`/`cbz`) with unconditional or NOP. Reflash with `openocd ... program firmware.bin verify reset exit 0x08000000`.

## Why it matters
Concrete CLI workflow — copy-pasteable commands for both dumping and patching. Pairs well with [[wrongbaud-ghidra-stm32-loader]] for the Ghidra crowd vs radare2 crowd.

## Confidence
**medium** — single author, but commands are verifiable and consistent with OpenOCD documentation.
