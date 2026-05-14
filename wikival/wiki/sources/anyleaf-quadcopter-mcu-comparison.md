---
type: source
title: "AnyLeaf Quadcopter MCU Comparison"
source_type: blog
status: ingested
source_url: https://www.anyleaf.org/blog/quadcopter-flight-controller-mcu-comparison
author: AnyLeaf
confidence: high
created: 2026-05-12
updated: 2026-05-12
tags: [source, blog, mcu, comparison, anyleaf, stm32, h7-recommended]
---

# AnyLeaf Quadcopter MCU Comparison

## Summary

AnyLeaf is a small FPV hardware company that designs and sells their
own flight controllers. Their MCU comparison blog post is unusually
candid for a vendor — it openly recommends MCUs whose chips AnyLeaf
*doesn't* sell when they think those are objectively better.

The post is the most aggressive "F7 is obsolete" voice among the sources
consulted. Treat its tone as opinion, but its facts check out.

## Key Claims Extracted

- F4: 84–180 MHz Cortex-M4F, 512 KB – 1 MB flash, 128–256 KB RAM.
- F7: 216 MHz Cortex-M7, 512 KB – 1 MB flash, 256 KB – 512 KB RAM.
- F7 is "pin-compatible with F4, and designed to be a faster version of
  it" — explains why F405→F722 board redesigns were cheap.
- H7: 480–550 MHz Cortex-M7, 128 KB – 2 MB flash, 1 MB RAM.
- The article asserts F7s "have been replaced by H7" and "are not
  suitable for use in new FC designs."

> [!warn] Editorial tone
> AnyLeaf's "F7 obsolete" framing is stronger than what Betaflight's own
> Manufacturer Design Guidelines say. The official BF position is that
> F7 is supported, but limited to 4 motors and superseded by H7 for new
> designs — not that it's obsolete.

## What It Contributes to the Wiki

Adds independent verification of the F4 / F7 / H7 spec table and
contributes the "pin-compatible" claim. Cited from:

- [[STM32F722]]
- [[STM32 MCU Family in Betaflight]]

## Confidence

**high** for the spec table. **medium** for the editorial claims about
F7 being unsuitable for new designs (vendor opinion, not BF policy).

## Provenance

Fetched 2026-05-12.
