---
type: source
source_type: tool
title: "SVD-Loader — Simplifying Bare-Metal ARM Reverse Engineering"
author: "leveldown-security / h2lab"
url: https://github.com/leveldown-security/SVD-Loader-Ghidra
confidence: high
fetched: 2026-05-12
tags: [source, ghidra, svd, reverse-engineering]
---

# SVD-Loader (Ghidra plugin)

## What it does
Parses a vendor-supplied **SVD** (System View Description, an XML standard from ARM) file and auto-generates Ghidra memory blocks for every peripheral on the chip. With it loaded, an access to `0x40023C04` is no longer a magic number — Ghidra labels it `RCC_CR.PLLON` (for example).

## Requirements
- Cortex-M target (CM0/CM3/CM4/CM7 supported)
- Little-endian
- SVD file for the exact MCU (ST distributes them per family; arm.com mirrors them too)

## Why it matters
Manually mapping STM32F7 peripherals takes hours. SVD-Loader does it in seconds. Once peripheral registers are named, recognising **what** the firmware is doing (UART setup, timer config, GPIO toggle) becomes mechanical.

## Cross-references
- The vendor's SVD for the chip is the same XML CMSIS uses to generate `stm32f7xx.h` header files in the build process — same source of truth, different consumer.

## Confidence
**high** — widely used in the firmware-RE community; tracked on GitHub with multiple active forks (h2lab IDA port, wejn RP2040 fork).
