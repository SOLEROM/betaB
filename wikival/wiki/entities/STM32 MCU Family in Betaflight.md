---
type: entity
title: "STM32 MCU Family in Betaflight"
entity_type: mcu-family
status: active
vendor: STMicroelectronics
created: 2026-05-12
updated: 2026-05-12
tags: [entity, mcu, stm32, family, betaflight-target, hardware]
---

# STM32 MCU Family in Betaflight

## What It Is

The **STM32 Cortex-M** family from STMicroelectronics is the dominant
MCU lineage in the Betaflight ecosystem. Every BF-supported FC released
between roughly 2015 and 2026 has run on an STM32 part — with a recent
sliver of competition from the pin-compatible **Artery AT32F435** and
experimental builds for **Raspberry Pi RP2350**. (Source:
[[DeepWiki Betaflight Config MCU Families]], confidence: **high**.)

## Supported STM32 Series (BF 4.4 / 4.5 era, 2026)

| Series | Core | Clock | Flash | RAM | BF role | Status |
|--------|------|-------|-------|-----|---------|--------|
| **F411** | Cortex-M4F | 100 MHz | 512 KB | 128 KB | Budget AIO / whoop | Deprecated for new designs |
| **F405** | Cortex-M4F | 168 MHz | 1 MB | 192 KB | The workhorse | Supported, **4-motor max** for new designs |
| **G473 / G491 / G474** | Cortex-M4F | 170 MHz | 512 KB | 128 KB | Newer mainstream | Supported |
| **F722** ([[STM32F722]]) | Cortex-M7 | 216 MHz | 512 KB / 1 MB | 256 KB | Mid-range | Supported, **4-motor max** for new designs |
| **F745** | Cortex-M7 | 216 MHz | 1 MB | 320 KB | Mid-range, more flash | Supported, **4-motor max** for new designs |
| **F765** | Cortex-M7 | 216 MHz | 2 MB | 512 KB | High-end F7 | Supported |
| **H723 / H743** | Cortex-M7 | 480–550 MHz | 1–2 MB | 1 MB | High end | **Recommended for new designs** |
| **H56x / H72x / H73x / H74x** | Cortex-M7 | 480+ MHz | — | — | High end | Recommended |

> [!gap] Sourcing note
> The H5/H72x rows are quoted from [[Betaflight Manufacturer Design
> Guidelines]] which lumps several H7 sub-families together without
> giving individual specs. Confidence on the exact part numbers used in
> production is **medium**.

## Dropped Series (Historical)

| Series | Core | When dropped from BF | Reason |
|--------|------|----------------------|--------|
| F1 (STM32F103) | M3 | 2017 | Too little flash/RAM; M3 lacks FPU |
| F3 (STM32F303) | M4 | 2019 | Insufficient flash for modern BF features |
| F2 | M3 | Never officially supported | — |

(Source: [[AnyLeaf Quadcopter MCU Comparison]], confidence: **high**.)

## Selection Principles

A **rule of thumb** that recurs across every source consulted:

> Generally, the higher the number, the more powerful the MCU.
> *(Source: [[Betaflight Getting Started Hardware]].)*

But beyond raw power, the practical selection criteria are:

1. **Motor count.** F4 and F7 are **capped at 4 motors** for new designs
   by official Betaflight policy. Hex / octo / X8 builds must use H7.
2. **Hardware UART inversion.** F4 lacks it; F7 and H7 have it. Pilots
   on inverted protocols (FrSky SBUS, SmartPort) want F7/H7 for
   plug-and-play.
3. **Flash budget.** Modern BF (4.4+) is tight on the 512 KB F722
   without the [[Cloud Build System]]. F411 is even tighter and is
   deprecated for new designs.
4. **Cost.** F405 stays the volume leader because it is the cheapest
   "modern enough" option.
5. **Headroom for the future.** H7 is the only family with comfortable
   margin for the next 2–3 years of BF feature growth.

## Build-System Layout

The Betaflight build organizes MCU code by family under
`make/mcu/<FAMILY>.mk`:

```
make/mcu/
  STM32F4.mk    # F405, F411
  STM32F7.mk    # F722, F745, F765
  STM32G4.mk    # G473, G491, G474
  STM32H7.mk    # H723, H743, H750
  AT32F4.mk     # AT32F435 (Artery)
```

Each `.mk` file declares the device-specific `-D` define
(e.g., `-DSTM32F722xx`), the linker script (e.g.,
`stm32_flash_f722.ld`), startup file, HAL/CMSIS paths, and any
compiler-flag overrides per flash size. (Source: search-result
extraction from `betaflight/make/mcu/STM32F7.mk`; direct fetch was
unsuccessful, see Open Questions.)

The `betaflight/config` repository then has one config per board under
`configs/<BOARD>/config.h`, with an `FC_TARGET_MCU` directive selecting
the family. (Source: [[DeepWiki Betaflight Config MCU Families]],
confidence: **high**.)

## Non-STM32 Alternatives

- **AT32F435** (Artery, China). Pin-compatible with F405. Officially
  supported as a build target. Adds DMAMUX (more flexible peripheral
  routing). Started gaining traction during STM32 supply-chain shortages
  ~2022–2023.
- **Raspberry Pi RP2350** (dual Cortex-M33). Experimental Betaflight
  builds exist; not a mainstream FC platform as of 2026.

## Confidence Summary

- **high**: BF officially supports F411, F405, G4XX, F722, F745, F765,
  H7XX, AT32F435.
- **high**: F4 and F7 capped at 4 motors for new designs.
- **medium**: Exact "recommended / not recommended" status of each
  specific part (draft list in BF issue #197).
- **low**: When BF will drop F4 / F7 entirely.

## Related

- [[STM32F722]]
- [[Betaflight MCU Targets]]
- [[Cloud Build System]]

## Sources

- [[Betaflight Manufacturer Design Guidelines]]
- [[Betaflight Issue 197 Supported MCUs]]
- [[Oscar Liang FC Processors]]
- [[Oscar Liang Betaflight 4.4]]
- [[AnyLeaf Quadcopter MCU Comparison]]
- [[DeepWiki Betaflight Config MCU Families]]
- [[Betaflight Getting Started Hardware]]
