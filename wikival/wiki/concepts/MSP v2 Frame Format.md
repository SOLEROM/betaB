---
type: concept
title: "MSP v2 Frame Format"
status: documented
created: 2026-05-14
updated: 2026-05-14
tags: [concept, msp, protocol, frame, crc]
related:
  - "[[MSP Protocol]]"
  - "[[MSP DisplayPort]]"
  - "[[MSP over CRSF]]"
---

# MSP v2 Frame Format

## Overview

MSP v2 is the modern Betaflight wire format. It replaces MSP v1's 8-bit
command ID, 8-bit size, and weak XOR checksum with **16-bit IDs, 16-bit
size, and a real CRC** (`crc8_dvb_s2`). Betaflight uses V2 for everything
new; V1 stays for backward compatibility with old configurators and legacy
clients.

## Byte Layout

```
$ X type flags func_lo func_hi size_lo size_hi  payload[size]  crc
0 1  2    3      4       5       6       7      8 … 7+size    8+size
```

| Offset | Field | Size | In CRC? | Notes |
|--------|-------|------|---------|-------|
| 0 | `$` (0x24) | 1 | no | Frame-start preamble |
| 1 | `X` (0x58) | 1 | no | V2 indicator (V1 uses `M`) |
| 2 | direction | 1 | no | `<` request, `>` response, `!` error |
| 3 | flags | 1 | **yes** | bit 0 = NO_REPLY, bit 1 = ILMI |
| 4–5 | function | 2 | **yes** | 16-bit message ID, little-endian |
| 6–7 | size | 2 | **yes** | Payload length, little-endian (0–65535) |
| 8 … | payload | size | **yes** | Message data (little-endian fields) |
| last | checksum | 1 | — | `crc8_dvb_s2` over flags+func+size+payload |

(Source: [[inav-wiki-msp-v2]])

## Checksum: crc8_dvb_s2

```c
uint8_t crc8_dvb_s2(uint8_t crc, uint8_t a) {
    crc ^= a;
    for (int i = 0; i < 8; ++i)
        crc = (crc & 0x80) ? (crc << 1) ^ 0xD5 : (crc << 1);
    return crc;
}
```

- Polynomial: `0xD5` (DVB-S2 satellite standard).
- Init value: `0`.
- Covers bytes 3 through `7+size` (flags onward, excluding the preamble
  and direction).

This is materially stronger than V1's XOR sum — XOR misses all
swap-and-flip errors, while a real CRC catches them.

## Flags

| Bit | Name | Meaning |
|-----|------|---------|
| 0 | `NO_REPLY` | Sender does not want a response (fire-and-forget) |
| 1 | `ILMI` | In-Line Message Identifier — used by some radio link layers |

Other bits are reserved.

## Backwards Compatibility: V2-in-V1 Wrapping

A V2 message can be carried inside a V1 frame for clients that only speak V1:

1. Wrap the V2 message body (everything from the flags byte onward, minus
   the `$X` preamble) as the V1 payload.
2. Set the V1 command field to **255** (`MSP_V2_FRAME` —
   defined in `msp_protocol.h`).
3. Compute the V1 XOR sum normally.

V1-only parsers see an unknown command 255 and ignore it cleanly instead of
erroring on the unexpected framing. Newer parsers detect command 255 and
unpack the V2 payload.

(Source: [[inav-wiki-msp-v2]], [[betaflight-msp-protocol-h]])

## Why It Matters

- **65535 IDs** instead of 255 → room for vendor-specific groups
  (`MSP2_BETAFLIGHT_*` start at `0x3000`, `MSP2_COMMON_*` at `0x1000`,
  `MSP2_SENSOR_*` at `0x1F00`).
- **65535-byte payload** → no more JUMBO-frame hacks; OSD font uploads,
  blackbox blob reads, large config dumps all fit.
- **CRC** → reliable over noisy links (CRSF, ELRS, USB-CDC with line noise).

## Comparison with V1

| Property | MSP v1 | MSP v2 |
|----------|--------|--------|
| Preamble | `$M` | `$X` |
| Command ID width | 8 bits | 16 bits |
| Payload size width | 8 bits | 16 bits |
| Max payload | 255 bytes | 65535 bytes |
| Checksum | XOR | `crc8_dvb_s2` |
| Direction bytes | `<` `>` `!` | `<` `>` `!` (same) |

## Sources

- [[inav-wiki-msp-v2]]
- [[betaflight-msp-protocol-h]]
- [[betaflight-deepwiki-msp]]
