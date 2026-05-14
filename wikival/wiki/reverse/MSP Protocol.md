---
type: protocol
title: "MSP Protocol"
status: developing
direction: bidirectional
transport: serial, USB
created: 2026-05-12
updated: 2026-05-12
tags: [protocol, reverse, msp, serial, companion-computer]
---

# MSP Protocol

## Overview

MultiWii Serial Protocol (MSP) is Betaflight's primary binary communication protocol. It runs over UART or USB serial and allows a host (configurator, companion computer, OSD chip) to read telemetry from the FC and write control commands to it.

Two versions exist: **MSP v1** (legacy, 8-bit size field, max 255 bytes payload) and **MSP v2** (16-bit size, extended IDs). Most tools use v1 for standard commands.

## Frame Format (MSP v1)

```
'$' 'M' direction size cmd payload... checksum
```

| Field     | Size    | Notes |
|-----------|---------|-------|
| `$`       | 1 byte  | Preamble byte 1 |
| `M`       | 1 byte  | Preamble byte 2 (MultiWii) |
| direction | 1 byte  | `<` = host→FC request, `>` = FC→host response, `!` = error |
| size      | 1 byte  | Payload length in bytes (0–255) |
| cmd       | 1 byte  | Command ID |
| payload   | N bytes | Command-specific data (little-endian) |
| checksum  | 1 byte  | XOR of size + cmd + all payload bytes |

## Checksum / CRC

```
checksum = size XOR cmd XOR payload[0] XOR payload[1] ... XOR payload[N-1]
```

## Command Table (Autonomy-Relevant)

| ID (dec) | ID (hex) | Name | Direction | Payload | Notes |
|----------|----------|------|-----------|---------|-------|
| 106 | 0x6A | MSP_RC | FC→Host | 32 bytes (16 × uint16) | Current RC channel values, 1000–2000 |
| 200 | 0xC8 | MSP_SET_RAW_RC | Host→FC | 32 bytes (16 × uint16) | Inject synthetic RC values; requires [[MSP Override Mode]] active |
| 110 | 0x6E | MSP_ANALOG | FC→Host | 7 bytes | Battery voltage (0.1V), mAh, rssi, current |
| 109 | 0x6D | MSP_ALTITUDE | FC→Host | 6 bytes | Baro altitude (cm, int32) + vario (cm/s, int16) |

> [!key-insight] MSP_SET_RAW_RC is the autonomy primitive
> Sending `MSP_SET_RAW_RC` injects synthetic stick values that BF treats as real RC input — but only for channels listed in `msp_override_channels_mask`. The FC's PID loop, filtering, and failsafe all continue to operate normally.

## MSP_SET_RAW_RC Payload Format

```
uint16 ch1, ch2, ch3, ..., ch16   (little-endian, values 1000–2000)
```

Total: 32 bytes (16 channels × 2 bytes). All 16 channels must be sent even if only a subset is being overridden. Non-overridden channels fall through to the normal RC input.

## Channel Mask

The `msp_override_channels_mask` CLI parameter is a bitmask (bit N = channel N+1):

```
set msp_override_channels_mask = 47   # 0b00101111 = CH1,2,3,4,6
```

| Bit | Channel | Typical Assignment |
|-----|---------|-------------------|
| 0   | CH1     | Roll |
| 1   | CH2     | Pitch |
| 2   | CH3     | Throttle |
| 3   | CH4     | Yaw |
| 4   | CH5     | AUX1 (ARM) |
| 5   | CH6     | AUX2 (drop servo / mode) |

## Implementation in BF

- Source: `src/main/msp/msp.c`
- MSP OVERRIDE channel injection: `src/main/rx/msp.c`
- Parser entry point: `mspFcProcessCommand()`

## Related

- [[MSP Override Mode]] — BF feature that enables MSP_SET_RAW_RC to take effect
- [[Companion Computer]] — typical host using this protocol
- [[Aocoda F460 Stack]] — example hardware running BF with MSP over USB

## Gaps

> [!gap] MSP v2 format unverified
> MSP v2 extends the frame with 16-bit command IDs and 16-bit size. Full frame format not yet documented here.

> [!gap] MSP_ALTITUDE byte layout unverified
> Claimed: int32 altitude (cm) + int16 vario (cm/s) = 6 bytes. Needs verification against BF source.

## Sources

- [[FPV Autonomous Operation with Betaflight and Raspberry Pi]]
