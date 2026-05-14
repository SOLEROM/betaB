---
type: source
title: "iNav Wiki — MSP V2"
source_type: wiki
author: iNav project (community)
date_published: 2017+
url: https://github.com/iNavFlight/inav/wiki/MSP-V2
confidence: high
created: 2026-05-14
updated: 2026-05-14
tags: [source, msp, protocol, inav]
key_claims:
  - "MSP v2 uses $X header, 16-bit message IDs (65535 max), 16-bit payload size, crc8_dvb_s2 checksum"
  - "Two flags defined: NO_REPLY (bit 0) and ILMI (bit 1)"
  - "V2 messages can be tunneled inside V1 frames using function code 255 for backwards compatibility"
---

# iNav Wiki — MSP V2

## Summary

The iNav project's wiki page on MSP V2 is the canonical specification for the
$X-prefixed binary frame format. Betaflight implements the same spec (the two
projects share protocol DNA from the original MultiWii/Cleanflight fork chain),
so this wiki is treated as authoritative for Betaflight MSP v2 as well.

## What it contributes

- Byte-by-byte frame layout for the standard MSP v2 frame.
- The `crc8_dvb_s2` algorithm in C, with polynomial `0xD5`, init zero, covering
  flags + function + size + payload.
- The two flag bits (NO_REPLY, ILMI) — the only flag bits defined in V2.
- Backwards-compatibility wrapping: legacy parsers see a V1 frame with
  function code **255** (= `MSP_V2_FRAME` in Betaflight headers), payload
  carries the full V2 message minus the `$X` preamble.
- Recommendation that V2 should be preferred — V1 XOR checksum is weak.

## Cited from

- `inav/wiki/MSP-V2` page on GitHub.

## Confidence

**High.** Cross-checked against Betaflight `msp_serial.c` parser (same crc
function, same field order, same wrap-in-V1 logic).

## Related

- [[MSP v2 Frame Format]]
- [[MSP Protocol]]
- [[Research - MSP Protocol Controlling Betaflight Firmware]]
