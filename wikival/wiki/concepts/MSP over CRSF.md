---
type: concept
title: "MSP over CRSF"
status: documented
created: 2026-05-14
updated: 2026-05-14
tags: [concept, msp, crsf, expresslrs, wireless-config, tunneling]
related:
  - "[[MSP Protocol]]"
  - "[[ExpressLRS]]"
  - "[[Betaflight Configurator]]"
---

# MSP over CRSF

## Overview

MSP-over-CRSF tunnels MSP frames through the CRSF (Crossfire) telemetry
link, so Betaflight Configurator can talk to a flight controller **wirelessly**
via an ExpressLRS or TBS Crossfire receiver — no USB cable.

The transport is asymmetric: large reads (FC → goggles) get 58-byte chunks;
writes (goggles → FC) get only 8-byte chunks because of OpenTX/EdgeTX's
outbound telemetry buffer limit.

(Source: [[betaflight-crsf-protocol-h]])

## CRSF Frame Types

| Frame type | ID | Direction | Chunk size | Purpose |
|------------|-----|-----------|------------|---------|
| `CRSF_FRAMETYPE_MSP_REQ` | `0x7A` | Host → FC | small | Request — uses MSP sequence number as command identifier |
| `CRSF_FRAMETYPE_MSP_RESP` | `0x7B` | FC → Host | **58 bytes** | Response, chunked |
| `CRSF_FRAMETYPE_MSP_WRITE` | `0x7C` | Host → FC | **8 bytes** | Write, chunked |

(Source: [[betaflight-crsf-protocol-h]])

## Why Chunking

A CRSF frame's payload is at most 60 bytes (header overhead eats the rest
of the 64-byte frame). MSP frames are commonly larger (an OSD font upload
is hundreds of KB). The MSP body is split across multiple CRSF frames and
reassembled at the other end.

Outbound writes from the radio handset suffer additionally because OpenTX's
telemetry buffer caps outbound chunks at **8 bytes**, so big writes (OSD
char upload, large config blobs) take many frames.

## Topology

```
  Configurator (PC / phone)
        │ USB-CDC or WiFi/TCP
        ▼
  Radio handset (OpenTX/EdgeTX) ──┐
        │ CRSF over UART          │ (MSP frame fragmented into 0x7A / 0x7C chunks)
        ▼                         │
  TX module (ELRS / Crossfire)    │
        │ 2.4 GHz / 915 MHz       │
        ▼                         │
  RX module (ELRS / Crossfire) ←──┘
        │ CRSF over UART
        ▼
  Flight controller (Betaflight) — runs the normal MSP parser
```

ExpressLRS additionally supports MSP-over-CRSF-over-WiFi/TCP, letting
Configurator connect to a TX-side web service that bridges to the FC.

## Use Cases

- Tune PIDs at the field without a laptop tethered to the quad.
- Read blackbox traces over the radio link (slow but works).
- Adjust VTX channel/power when the FC USB port is hard to reach.

## Pitfalls

- **Large reads** (e.g. `MSP_OSD_CHAR_READ` for every font character) can
  take minutes because each char is up to 64 bytes and ~30 CRSF telemetry
  slots per second is typical.
- **Chunk reassembly bugs**: known issue
  [betaflight#13847](https://github.com/betaflight/betaflight/issues/13847)
  where MSP responses over CRSF stalled at 2 chunks.
- **Sequence numbering**: the request's MSP sequence is reused as part of
  the CRSF frame; out-of-order chunks corrupt reassembly.

## Implementation in BF

- `src/main/telemetry/crsf.c` — handles CRSF telemetry and MSP framing.
- `src/main/rx/crsf.c` — handles CRSF RX side.
- `src/main/rx/crsf_protocol.h` — frame type constants.

## Sources

- [[betaflight-crsf-protocol-h]]
- [[betaflight-deepwiki-msp]]

## Gaps

> [!gap] Sequence/status byte layout undocumented
> The exact byte layout of the MSP sequence and status fields inside
> `CRSF_FRAMETYPE_MSP_REQ` payload is not in the protocol header — would
> need to read the parser in `crsf.c` to confirm.
