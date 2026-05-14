---
noteId: "d1fa24804f9c11f194a2c3b1eecd91b7"
type: protocol
title: "Blackbox Format"
status: stub
direction: fc-to-host
transport: SD card | serial | OpenLog
created: 2026-05-14
updated: 2026-05-14
tags: [protocol, reverse, blackbox, logging, stub]
---

# Blackbox Format

## Overview
<!-- Binary log format Betaflight writes during flight: field encoding (delta + huffman-like), header schema, decoder pipeline (blackbox_decode / log-viewer). -->

> [!gap]
> Stub. Pending: header layout, predictor table, frame types (I/P/G/S/E/H), decoder source walk.

## Implementation in BF
- `src/main/blackbox/blackbox.c`, `blackbox_encoding.c`, `blackbox_io.c`
- Field schema: `blackbox_fielddefs.h`

## Sources
-
