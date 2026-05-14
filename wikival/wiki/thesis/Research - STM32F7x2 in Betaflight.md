---
type: synthesis
title: "Research: STM32F7x2 in Betaflight"
created: 2026-05-12
updated: 2026-05-12
tags: [research, synthesis, stm32, mcu, f7x2, f722, betaflight]
status: developing
research_query: "stm32f7x2 betaflight flight controller"
related:
  - "[[STM32F722]]"
  - "[[STM32 MCU Family in Betaflight]]"
  - "[[Betaflight MCU Targets]]"
  - "[[Cloud Build System]]"
sources:
  - "[[Oscar Liang FC Processors]]"
  - "[[Oscar Liang Betaflight 4.4]]"
  - "[[AnyLeaf Quadcopter MCU Comparison]]"
  - "[[Betaflight Manufacturer Design Guidelines]]"
  - "[[Betaflight Issue 197 Supported MCUs]]"
  - "[[Betaflight Configurator Issue 3366 F7X2 Missing]]"
  - "[[DeepWiki Betaflight Config MCU Families]]"
  - "[[Flying Rabbit Creating BF Target]]"
  - "[[Betaflight Getting Started Hardware]]"
---

# Research: STM32F7x2 in Betaflight

## Overview

The **STM32F7x2** is the BF target family containing the **STM32F722**,
the second-most-popular Betaflight MCU after the F405. This synthesis
covers the F7x2 / F722 substantively, including the Betaflight target
system around it.

## Key Findings

### 1. The Betaflight target is `STM32F7X2`
A single binary target named `STM32F7X2` (sometimes `F7X2RE`) covers
the **STM32F722RET6** (512 KB flash) and **STM32F722RGT6** (1 MB flash)
parts. The `X` is a wildcard digit. (Source: [[Flying Rabbit Creating
BF Target]] + [[Betaflight Configurator Issue 3366 F7X2 Missing]],
confidence: **high**.)

### 2. The F7 family is "supported but not preferred" for new designs
Per the [[Betaflight Manufacturer Design Guidelines]], F4 and F7 are
both **capped at 4 motor outputs for new FC designs** — H7 is required
for hex/octo/X8. F411 is explicitly deprecated for new designs. The H7
family (H56x/H72x/H73x/H74x) is the recommendation. (Source:
[[Betaflight Manufacturer Design Guidelines]], confidence: **high**.)

### 3. The 512 KB F722 RET6 is kept alive by the Cloud Build
Betaflight 4.4 introduced the **[[Cloud Build System]]** — a server-side
compile service that lets users tailor which drivers and features are
included. This is the binding mechanism that keeps the
flash-constrained F411 and F722RE viable as BF keeps growing. (Source:
[[Oscar Liang Betaflight 4.4]], confidence: **high**.)

### 4. F7 advantages over F4 that haven't been superseded
- 216 MHz Cortex-M7 with L1 cache vs 168 MHz Cortex-M4F
- Hardware UART signal inversion on all UARTs (plug-and-play SBUS /
  SmartPort)
- More UARTs typically routed
- No F4-style SPI1 DMA limitation
- Pin-compatible with F405 (cheap board redesigns)

(Sources: [[Oscar Liang FC Processors]], [[AnyLeaf Quadcopter MCU
Comparison]]; confidence: **high**.)

### 5. The unified-target system
Since BF 4.0, the project ships **one binary per MCU family** and stores
**per-board configs as text** in `betaflight/config`. The user's
Configurator picks the right binary for the MCU and applies the right
config for the board. This is the precondition that makes Cloud Build
possible. (Source: [[Flying Rabbit Creating BF Target]] +
[[DeepWiki Betaflight Config MCU Families]], confidence: **high**.)

## Key Entities

- [[STM32F722]] — the MCU itself, 216 MHz Cortex-M7, 512 KB / 1 MB flash
- [[STM32 MCU Family in Betaflight]] — the wider MCU landscape

## Key Concepts

- [[Betaflight MCU Targets]] — how target names work, why `STM32F7X2`
  covers multiple physical chips
- [[Cloud Build System]] — the BF 4.4 server-side compile service that
  rescues constrained flash budgets

## Contradictions

- **[[AnyLeaf Quadcopter MCU Comparison]]** says F7s "are not suitable
  for use in new FC designs."
- **[[Betaflight Manufacturer Design Guidelines]]** does **not** ban F7
  for new designs; it caps F7 at 4 motors and recommends H7. The F7 is
  still on the supported list.

The BF official doc is the authoritative source; AnyLeaf's stronger
"obsolete" framing reads as commercial editorializing (AnyLeaf sells H7
boards). The wiki adopts the BF official stance.

## Open Questions

> [!gap] Direct view of `STM32F7.mk`
> WebFetch failed (404) on the github.com/betaflight/betaflight blob URL
> and on the raw.githubusercontent.com URL — likely because the file
> exists in the master branch but the WebFetch crawler hit a redirect or
> rate limit. The build-system claims (define `-DSTM32F722xx`, linker
> script `stm32_flash_f722.ld`, `OPTIMISE_SPEED→SIZE` flash trick) are
> from search-result snippets and need direct file verification.

> [!gap] Compiler-flag policy for 512 KB targets
> Whether the `OPTIMISE_SPEED → OPTIMISE_SIZE` override is permanent for
> F722RE or only triggered when the build would overflow is not yet
> verified.

> [!gap] Per-MCU "recommended for new designs" status
> The [[Betaflight Issue 197 Supported MCUs]] draft list (F722RGT6
> recommended, F722RET6 not recommended) is **not** ratified policy. The
> formal stance lives in the manufacturer guidelines, which doesn't
> split RE vs RG.

> [!gap] AT32F435 Configurator target name
> The exact target string used by AT32 boards in Configurator wasn't
> captured in any source consulted.

> [!gap] When BF will drop F4 / F7
> No source provides a roadmap date. The trajectory (deprecate F411,
> cap F4/F7 at 4 motors, recommend H7) suggests a multi-year wind-down
> but the timeline is speculation.

## Sources

- [[Oscar Liang FC Processors]] — Oscar Liang, continuously updated,
  high confidence
- [[Oscar Liang Betaflight 4.4]] — Oscar Liang, ~2023, high confidence
- [[AnyLeaf Quadcopter MCU Comparison]] — AnyLeaf blog, high confidence
  on specs, medium on editorial
- [[Betaflight Manufacturer Design Guidelines]] — BF official, high
  confidence
- [[Betaflight Issue 197 Supported MCUs]] — BF docs repo issue, medium
  confidence (draft list, not ratified)
- [[Betaflight Configurator Issue 3366 F7X2 Missing]] — BF Configurator
  issue, 2023-03-03, medium confidence
- [[DeepWiki Betaflight Config MCU Families]] — auto-generated docs,
  medium confidence
- [[Flying Rabbit Creating BF Target]] — blog, 2020-11-07, medium
  confidence (somewhat dated)
- [[Betaflight Getting Started Hardware]] — BF official, high confidence

## Provenance

Autoresearch session 2026-05-12. Rounds: 2 (broad + gap-fill).
Web searches: 8. Pages fetched: 7 (1 failed: STM32F7.mk).
