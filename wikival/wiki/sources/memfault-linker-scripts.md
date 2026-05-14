---
type: source
source_type: blog
title: "From Zero to main(): Demystifying Firmware Linker Scripts"
author: "Tyler Hoffman (Memfault / Interrupt)"
date_published: 2019-06-04
url: https://interrupt.memfault.com/blog/how-to-write-linker-scripts-for-firmware
confidence: high
fetched: 2026-05-12
tags: [source, build-system, linker, cortex-m]
---

# Memfault — Linker Scripts for Firmware

## What it covers
Step-by-step deconstruction of an ARM Cortex-M linker script using the SAMD21G18 as the example. Explains MEMORY blocks, SECTIONS, the LMA/VMA dual-address dance for `.data`, and how the linker injects the symbols (`_sdata`, `_edata`, `_sbss`, `_ebss`) that the reset handler consumes.

## Key claims
- **The linker script is the blueprint** — it tells `ld` where every section goes (flash vs. SRAM).
- **MEMORY** describes regions: flash at `0x00000000` 256 KB, SRAM at `0x20000000` 32 KB (SAMD21 example; STM32 flash is at `0x08000000`).
- **`KEEP(*(.vectors))`** at start of `.text` pins the vector table to flash offset 0 so the reset handler pointer lands at `+0x04`.
- **`.bss` uses `(NOLOAD)`** — zero-init region occupies no bytes in the binary image.
- **`.data` has LMA ≠ VMA** — initialiser values are stored in flash (LMA) but the variables live in RAM (VMA); reset handler copies LMA→VMA before `main()`.
- **Symbol resolution** — before linking, `main` is `0x00000000`; after, it gets a real address (e.g. `0x00000294`).
- **Final output is `.elf`** — `objcopy` then converts to `.bin` or `.hex` for the programmer.

## Why it matters for the BF wiki
Defines the build foundation that produces every `betaflight_X_Y_Z_STM32F7X2.hex` file. Any time we patch a binary or analyse a dump, we are working with the artifact this script produced. Critical for [[ARM Cortex-M Firmware Build Process]] and [[Linker Script]].

## Confidence
**high** — Memfault is an embedded-firmware product company; the post is canonical reference material reused across many embedded blogs.
