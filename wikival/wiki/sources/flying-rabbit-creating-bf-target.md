---
type: source
title: "Flying Rabbit Creating BF Target"
source_type: blog
status: ingested
source_url: https://flying-rabbit-fpv.com/2020/11/07/creating-a-betaflight-target/
author: Flying Rabbit FPV
date_published: 2020-11-07
confidence: medium
created: 2026-05-12
updated: 2026-05-12
tags: [source, blog, unified-target, diy-fc, betaflight-target, stm32f722]
---

# Flying Rabbit Creating BF Target

## Summary

A 2020 walkthrough of creating a Betaflight target for a DIY flight
controller (built around an STM32F722). Written after Betaflight 4.0
moved to **unified targets** — i.e., one binary per MCU plus a config
text file per board.

The post is dated but is the most explicit "step by step" public guide
on:

- Naming convention for the MCU target
- The shift from `target.h` / `target.c` / `target.mk` to a single
  `.config` file
- How to submit a custom board's config to the BF Configurator

## Key Claims Extracted

- The unified target name for an STM32F722 is **"STM32F7x2"** (read as
  "any F7-series 7x2 family part").
- *"With the unified target system, there is only one binary for each
  type of STM32."*
- Board-specific I/O mapping is via CLI commands in a `.config` text
  file, not compiled into the binary.
- Custom board configs are submitted via pull request to be included in
  Configurator.

## What It Contributes to the Wiki

The conceptual model behind unified targets and the F7x2 naming
convention. Cited by:

- [[Betaflight MCU Targets]]
- [[STM32F722]]

## Confidence

**medium**. The post is from 2020 (over 5 years old as of 2026-05-12)
and predates the Cloud Build system and several MCU policy changes. The
unified-target description still holds, but specific tooling references
may be stale.

## Caveats

The 2020 publication date means anything about specific Configurator
versions, build scripts, or PR processes should be re-verified against
current BF docs before relying on it operationally.

## Provenance

Fetched 2026-05-12.
