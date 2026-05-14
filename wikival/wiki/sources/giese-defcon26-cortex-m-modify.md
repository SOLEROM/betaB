---
type: source
source_type: conference-talk
title: "How to Modify ARM Cortex-M Based Firmware (DEFCON 26 IoT Village)"
author: "Dennis Giese"
date_published: 2018-08
url: https://dontvacuum.me/talks/DEFCON26-IoT-Village/DEFCON26-IoT-Village_How_to_Modify_Cortex_M_Firmware-Xiaomi.pdf
secondary_url: https://hackaday.com/2019/10/19/customizing-xiaomi-arm-cortex-m-firmware/
confidence: high
fetched: 2026-05-12
tags: [source, reverse-engineering, binary-patching]
---

# Giese DEFCON 26 — Modifying Cortex-M Firmware

## What it covers
End-to-end methodology for changing the behaviour of a Cortex-M device when you do not have the source. Worked example: Xiaomi IoT cameras / vacuums.

## Methodology (generic to any Cortex-M target)
1. **Acquisition**
   - Dump SPI flash via JTAG/SWD, or with the chip desoldered.
   - Intercept firmware-update traffic in transit.
   - Pull from the vendor's update CDN.
2. **Parsing**
   - Proprietary container formats first need converting to ELF. Python scripts exist for most common vendor wrappers.
3. **Function identification**
   - Search for human-readable tag strings (`TAG_MAC`, `TAG_DID`, `TAG_KEY`) to anchor calibration / credential blobs.
   - Use *signature diffing* against a reference binary to recognise function bodies.
4. **Patching**
   - Carve out free flash space for new code (often the tail of `.text` or an unused vector slot).
   - Frameworks: **[[Nexmon]]** — C-based binary-patching framework with first-class Cortex-M support.
5. **Repackage + reflash**
   - Recompute the vendor checksum / signature.
   - Push back to the device via the same channel used to dump.

## Tools called out
- **IDA Pro** for disassembly.
- **Nexmon** for the actual patch insertion.
- **Python** glue for format conversion.

## Why it matters
Best public reference for the *no-source-code* attack path the user asked about. Directly answers part 2 of the research brief.

## Confidence
**high** — conference talk by an established IoT-security researcher, corroborated by Hackaday writeup.

## Note
The raw PDF resists WebFetch because the slides are mostly imagery. Hackaday article (`secondary_url`) is the readable summary used for extraction.
