---
noteId: "d207b9104f9c11f194a2c3b1eecd91b7"
type: protocol
title: "EEPROM Layout"
status: stub
direction: n/a
transport: internal flash
created: 2026-05-14
updated: 2026-05-14
tags: [protocol, reverse, eeprom, config, pg, stub]
---

# EEPROM Layout

## Overview
<!-- How Betaflight serializes configuration to flash: parameter group (`pg_`) macro system, per-PG versioning, validation, MSP `MSP_EEPROM_WRITE(250)` flow. -->

> [!gap]
> Stub. Pending: PG record format on flash, header/version bytes, write/erase sector mechanics, hash-based validation. Note: 4.5 sets `EEPROM_SIZE = 32768` in `target.h`.

## Related
- [[MSP Protocol]] — `MSP_EEPROM_WRITE`
- [[CLI Internals]] — `save`, `defaults`, `diff` all hit EEPROM

## Source anchors
- `src/main/config/config.c`, `config_streamer.c`, `pg/pg.c`, `parameter_group.h`

## Sources
-
