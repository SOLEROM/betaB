---
type: source
title: "Oscar Liang FC Processors"
source_type: blog
status: ingested
source_url: https://oscarliang.com/f1-f3-f4-flight-controller/
author: Oscar Liang
confidence: high
created: 2026-05-12
updated: 2026-05-12
tags: [source, blog, oscar-liang, mcu, stm32, comparison]
---

# Oscar Liang FC Processors

## Summary

Oscar Liang's long-running survey article on FPV flight-controller
processors. The page is continuously updated (the URL still says
`f1-f3-f4` but the body covers F4, G4, F7, H7, and AT32). It's the
single most-cited consumer-facing comparison of FC MCUs and is treated
across the FPV community as the canonical "which chip should I buy"
reference.

## Key Claims Extracted

- **F1 / F3 are no longer supported** by Betaflight (F1 dropped 2017,
  F3 dropped 2019).
- **F4 (STM32F405)** is "pretty much the most common option" — 168 MHz
  Cortex-M4F, 1 MB flash, 192 KB RAM.
- **F4 lacks hardware UART signal inversion** — needs an external
  inverter for FrSky inverted serial.
- **F7 (STM32F722)** is "the most popular F7 chip" — 216 MHz Cortex-M7,
  512 KB flash, 256 KB RAM.
- **F7 has hardware UART signal inversion on all UARTs** — plug-and-play
  for FrSky.
- **F7 / H7 support 8 kHz PID loop and DSHOT600**; F4 is capped around
  4 kHz / DSHOT300 in practice.
- **F4 / F7 are limited to 4 motor outputs** for new BF designs;
  hex/octo requires H7.
- **H7 (STM32H743)** is 480 MHz, 2 MB flash, 1 MB RAM — "by far the
  fastest processor available".
- **Cloud Build (BF 4.4)** keeps the F722's small flash workable.

## What It Contributes to the Wiki

The authoritative consumer-level summary of MCU choice. Used as a source
for:

- [[STM32F722]]
- [[STM32 MCU Family in Betaflight]]
- [[Betaflight MCU Targets]]

## Confidence

**high** — Oscar Liang is a long-running, reputation-driven FPV blogger
whose technical claims have been independently verified against ST
datasheets and Betaflight source code many times. The factual core
(clock speeds, flash sizes, UART counts) is uncontroversial.

## Caveats

- The article's framing is consumer-buyer, not engineer. Some
  trade-offs (e.g., why F4 has SPI1 DMA limits) are mentioned without
  citation.
- The URL slug (`f1-f3-f4`) is misleading; the page has been re-edited
  many times and now covers H7 and AT32 too. The publish date in any
  cite should be the most-recent-updated date, not the original.

## Provenance

Fetched 2026-05-12 during the autoresearch session on
"stm32f8x2 betaflight flight controller".
