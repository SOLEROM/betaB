---
type: index
title: "Reverse Engineering Index"
created: 2026-05-12
updated: 2026-05-12
tags: [index, reverse-engineering]
---

# Reverse Engineering

Protocol analysis, binary format documentation, and low-level Betaflight internals. Goal: understand what BF does without relying on official docs.

## Topics

| Topic                                           | Priority | Notes                                                                                                  |
| ----------------------------------------------- | -------- | ------------------------------------------------------------------------------------------------------ |
| [[MSP Protocol]]                                | HIGH     | MultiWii Serial Protocol — frame format, command table, versioning — **page exists**                   |
| [[MSP Commands Reference]]                      | HIGH     | Full table of MSP command IDs and their payloads                                                       |
| [[DSHOT Protocol]]                              | HIGH     | Bidirectional DSHOT frame format, telemetry encoding                                                   |
| [[CRSF Protocol]]                               | MED      | ExpressLRS/TBS Crossfire serial protocol                                                               |
| [[EEPROM Layout]]                               | MED      | How BF serializes config to flash — pg_ macro system                                                   |
| [[Blackbox Format]]                             | MED      | Binary log format, field encoding, decoder                                                             |
| [[Bootloader]]                                  | MED      | STM32 DFU + BF bootloader — how flashing works                                                         |
| [[Build System RE]]                             | LOW      | Make targets, target inheritance, unified targets                                                      |
| [[OSD Font Format]]                             | LOW      | MAX7456 character encoding                                                                             |
| [[CLI Internals]]                               | MED      | How CLI parsing works in the codebase                                                                  |
| [[Bin to Hex Conversion and Constant Patching]] | MED      | objcopy bin↔hex workflow + worked example finding/patching `CRSF_BAUDRATE` in a dump — **page exists** |

## MSP Quick Reference

MSP frames:
```
$M< (request)  or  $M> (response)  or  $M! (error)
[size][cmd][payload...][checksum]
```

Checksum: XOR of size + cmd + all payload bytes.

> [!gap] MSP v2
> MSP v2 has a different frame format with 16-bit command IDs. Need to document and compare.

## Key Files in Source

- `src/main/msp/msp.c` — MSP command dispatch
- `src/main/msp/msp_protocol.h` — command ID definitions
- `src/main/config/config.c` — EEPROM read/write
- `src/main/flight/pid.c` — PID loop (primary target for understanding)
