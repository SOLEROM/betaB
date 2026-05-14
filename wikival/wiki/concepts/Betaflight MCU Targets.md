---
type: concept
title: "Betaflight MCU Targets"
status: developing
created: 2026-05-12
updated: 2026-05-12
tags: [concept, build-system, target, mcu, unified-target, configurator]
---

# Betaflight MCU Targets

## What It Is

A **Betaflight target** is the binary identity of a firmware build: the
combination of MCU family, flash layout, and pin/peripheral mapping
that's baked into the flashed image.

Since Betaflight 4.0 the project moved to **unified targets**: one
binary per MCU family covers many physical FC boards, and the
board-specific resource map is loaded as a `config` (a text blob of CLI
commands applied at boot). (Source: [[Flying Rabbit Creating BF
Target]], confidence: **high**.)

## Naming Convention

Target names follow the pattern `STM32<FAMILY><WILDCARD><FLASH-SUFFIX>`:

| Target | Covers chips | Flash |
|--------|--------------|-------|
| `STM32F411` | F411CE / F411RE | 512 KB |
| `STM32F405` | F405RG | 1 MB |
| `STM32G4XX` | G473 / G491 / G474 | 512 KB |
| `STM32F7X2` (a.k.a. `F7X2RE`) | F722RE / F722RG | 512 KB / 1 MB |
| `STM32F745` | F745VG | 1 MB |
| `STM32F765` | F765VI | 2 MB |
| `STM32H743` | H743VI / H743VIH6 | 2 MB |
| `AT32F435` | AT32F435 | 1 MB |

`X` is a **wildcard digit** in the target name, so `STM32F7X2` reads as
"any STM32F7-series 7x2-family part". The `RE` suffix on `F7X2RE`
indicates the 512 KB flash package; the same binary loads on `RG` (1 MB)
parts. (Source: [[Flying Rabbit Creating BF Target]] + Configurator
issue #3366; confidence: **high**.)

## What a Target Defines

Three layers:

1. **MCU family** — selects which `make/mcu/STM32<X>.mk` to compile
   against, picks the linker script, sets `-D<chip>xx`, picks HAL/CMSIS
   driver versions.
2. **Flash layout** — picks the linker script (e.g., `stm32_flash_f722.ld`
   for 512 KB, `stm32_flash_f74x.ld` for 1 MB). Smaller flash variants
   may override compiler flags (`OPTIMISE_SPEED` → `OPTIMISE_SIZE`) to
   fit. (Source: search snippets of `STM32F7.mk`, confidence: **medium**.)
3. **Default peripheral set** — the binary includes drivers for a
   default superset of peripherals (gyros, OSDs, barometers, flash chips,
   ESC protocols). Board-specific selection happens at runtime through
   the `config` blob.

## Unified Target vs Legacy Target

Pre-4.0 Betaflight had one target per physical FC board: hundreds of
`target.h` / `target.c` / `target.mk` triplets in
`src/main/target/<BOARDNAME>/`. Each one hard-coded the pin assignments
for that specific FC.

From 4.0 onward, that file structure was collapsed:

- **One binary per MCU family** (`STM32F7X2`, `STM32H743`, …) lives in
  `betaflight/betaflight`.
- **One config per board** (`HGLRCF722AIO`, `MAMBAF722_2022B`, …) lives
  in `betaflight/config`. These are pure CLI text — pin mappings, DMA
  assignments, gyro orientation, OSD font choice, etc.
- The Configurator downloads the right binary for your MCU and the right
  config for your board, then applies the config via CLI.

This is why a single STM32F722 binary can be flashed to a Mamba F722, a
GEPRC F722-HD, a Kakute F722, a Foxeer F722 V2, and so on — each board
just applies a different config on top.

## Cloud Build Integration

The unified-target system is the foundation for the
[[Cloud Build System]] introduced in BF 4.4. Because the binary already
ships with a configurable peripheral surface, the cloud build can omit
unused drivers / features at compile time to fit a custom build into
constrained flash (especially the 512 KB F411 and F722). (Source:
[[Oscar Liang Betaflight 4.4]], confidence: **high**.)

## Implications for Builders

- **Don't memorize board-specific targets** — the Configurator picks
  the binary based on your MCU.
- **Custom DIY board?** You add a `config` text file (not a full target)
  and submit it via PR to `betaflight/config`. (Source: [[Flying Rabbit
  Creating BF Target]], confidence: **high**.)
- **Need a feature that's not in the default build?** Use the Cloud
  Build, enable Expert Mode, tick the driver, and re-flash.
- **F4 → F7 / F7 → H7 migration**: same MCU family means same binary;
  changing family means new binary, but the config (resource map) is
  largely portable if pin functions line up.

## Open Questions

> [!gap] Exact MCU flavor list for F7X2
> The author's draft list in [[Betaflight Issue 197 Supported MCUs]]
> labels `F722RET6` as "not recommended" but `F722RGT6` as "recommended"
> — yet they share a single binary target `STM32F7X2`. How the build
> system handles the flash-size difference (linker script selection
> at compile time) needs direct verification against `STM32F7.mk`.

> [!gap] AT32F435 target name
> The exact target string used by AT32F435 boards in Configurator is
> not captured in the sources consulted.

## Related

- [[STM32F722]]
- [[STM32 MCU Family in Betaflight]]
- [[Cloud Build System]]

## Sources

- [[Flying Rabbit Creating BF Target]]
- [[Betaflight Configurator Issue 3366 F7X2 Missing]]
- [[DeepWiki Betaflight Config MCU Families]]
- [[Oscar Liang Betaflight 4.4]]
- [[Betaflight Manufacturer Design Guidelines]]
