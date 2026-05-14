---
type: source
source_type: blog
title: "Disassembling a Cortex-M Raw Binary File with Ghidra"
author: "Feabhas (Sticky Bits)"
date_published: 2022-12
url: https://blog.feabhas.com/2022/12/disassembling-a-cortex-m-raw-binary-file-with-ghidra/
confidence: high
fetched: 2026-05-12
tags: [source, ghidra, reverse-engineering, cortex-m]
---

# Feabhas — Cortex-M Raw Binary in Ghidra

## Key claims
- **Language for Cortex-M4**: pick `ARM v7 little endian`. Ghidra cannot infer ABI from a raw binary.
- **Flash aliasing** — STM32 actually executes from `0x08000000` but the bootloader remaps to `0x00000000`. Default Ghidra import lands at `0x00000000` and disassembles incorrectly. Fix: set base to `0x08000000`.
- **Vector table** — Ghidra recognises offset 0 as MSP, offset 4 as Reset, etc.
- **Aggressive Instruction Finder** — enable in Analysis to improve Thumb code discovery. Thumb entry points are marked with the `+1` suffix on function addresses (LSB-set indicates Thumb mode).

## Why it matters
Second independent walkthrough confirming the [[Loading Cortex-M Firmware in Ghidra]] recipe. Highlights the *aliasing trap* that catches first-time analysts.

## Confidence
**high** — Feabhas is an embedded-training firm; their technical blog is reliable reference material.
