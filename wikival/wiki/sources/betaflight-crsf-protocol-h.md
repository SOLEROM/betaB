---
type: source
title: "Betaflight Source — crsf_protocol.h"
source_type: source-code
author: Betaflight project
date_published: 2025+
url: https://github.com/betaflight/betaflight/blob/master/src/main/rx/crsf_protocol.h
confidence: high
created: 2026-05-14
updated: 2026-05-14
tags: [source, msp, crsf, tunneling, wireless-config]
key_claims:
  - "CRSF_FRAMETYPE_MSP_REQ = 0x7A — MSP request over CRSF"
  - "CRSF_FRAMETYPE_MSP_RESP = 0x7B — reply chunked in 58-byte segments"
  - "CRSF_FRAMETYPE_MSP_WRITE = 0x7C — write chunked in 8-byte segments (OpenTX outbound limit)"
---

# Betaflight Source — crsf_protocol.h

## Summary

The CRSF frame-type enumeration. Three of these frame types form the
MSP-over-CRSF tunnel that lets ExpressLRS / TBS Crossfire receivers carry
Betaflight Configurator traffic wirelessly: the radio link from the
goggles/transmitter to the receiver to the FC becomes a transport for MSP
frames.

## What it contributes

- Numeric frame type IDs for the three MSP tunneling frames.
- The asymmetric chunk sizes — RESP is 58 bytes per chunk (CRSF frame
  payload limit), WRITE is only 8 bytes (limited by OpenTX's outbound
  telemetry buffer).
- This is why large MSP writes (OSD font upload, etc.) over CRSF can be
  slow or fail without careful chunking.

## Confidence

**High.** Primary source code.

## Related

- [[MSP over CRSF]]
- [[ExpressLRS]]
- [[MSP Protocol]]
