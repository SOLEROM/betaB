---
type: source
title: "Betaflight Issue 197 Supported MCUs"
source_type: github-issue
status: ingested
source_url: https://github.com/betaflight/betaflight.com/issues/197
author: Betaflight contributor (issue author)
confidence: medium
created: 2026-05-12
updated: 2026-05-12
tags: [source, github-issue, mcu, supported-mcus, draft-policy]
---

# Betaflight Issue 197 Supported MCUs

## Summary

A GitHub issue on the **betaflight/betaflight.com** documentation repo
requesting a **single canonical table of supported MCUs**. The issue
itself proves the gap: as of the issue's writing there was no single
source of truth on the BF docs site.

The author drafts a candidate table from multiple existing sources and
explicitly disclaims its authority.

## Key Claims Extracted (Draft List)

MCUs the author identifies as "mentioned as supported":

- STM32F405RGT6
- STM32F411CEU6
- STM32F722RET6
- STM32F722RGT6
- STM32F745VGT6
- STM32H743VIT6
- STM32H743VIH6

Proposed (not official) status categories:

- **Obsolete**: STM32F303CCT6
- **Not recommended for new designs**: F405, F411, F722RET6
- **Recommended for new designs**: F722RGT6, F745VGT6, H743 variants

## What It Contributes to the Wiki

- The exact factory part numbers for the F722 (RET6 vs RGT6) and F745
  (VGT6) variants used in BF-supported FCs.
- The first explicit hint that **within F722, the RET6 (512 KB) and
  RGT6 (1 MB) are treated differently** in the maintainers' minds.

Cited by:

- [[STM32F722]]
- [[STM32 MCU Family in Betaflight]]
- [[Betaflight MCU Targets]]

## Confidence

**medium**. The issue author explicitly says *"this table would need to
contain reliable information"* and that they pulled from "multiple more
or less reliable sources." The list reflects community consensus but
is **not** ratified policy.

The authoritative policy lives in [[Betaflight Manufacturer Design
Guidelines]], not this issue. Use this source to find part numbers, not
to make policy claims.

## Provenance

Fetched 2026-05-12.
