---
type: source
title: "DeepWiki Betaflight Config MCU Families"
source_type: third-party-docs
status: ingested
source_url: https://deepwiki.com/betaflight/config/3-mcu-families
author: DeepWiki (auto-generated from betaflight/config)
confidence: medium
created: 2026-05-12
updated: 2026-05-12
tags: [source, deepwiki, betaflight-config, mcu, repository-layout]
---

# DeepWiki Betaflight Config MCU Families

## Summary

DeepWiki is an **auto-generated documentation site** that analyzes
GitHub repos and produces structured docs. This page summarizes the
MCU family layout of the `betaflight/config` repository — the repo
that holds per-board `config.h` files for the unified-target system.

It's a useful index but, being machine-generated, can lag behind the
actual repo state.

## Key Claims Extracted

- Supported MCU families in `betaflight/config`: STM32F4, STM32F7,
  STM32H7, plus AT32F435 and (experimentally) Raspberry Pi RP2350.
- Repository layout: `configs/<BOARD_NAME>/config.h`.
- Board count is **not exhaustive** in the page; examples include
  `STELLARF4`, `STELLARF7`, `STELLARF7V2`, `HGLRCF722AIO`, `ARK_FPV`,
  `ACROSKYH743`, `FOXEERH743V2`.
- Each config has an `FC_TARGET_MCU` directive identifying the family.

## What It Contributes to the Wiki

The clearest external view of the unified-target repo layout. Used by:

- [[Betaflight MCU Targets]]
- [[STM32 MCU Family in Betaflight]]

## Confidence

**medium**. DeepWiki is auto-generated and unverified. For authoritative
layout claims, fall back to the repo itself
(github.com/betaflight/config).

## Provenance

Fetched 2026-05-12.
