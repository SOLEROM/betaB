---
noteId: "d20792014f9c11f194a2c3b1eecd91b7"
type: protocol
title: "CLI Internals"
status: stub
direction: bidirectional
transport: serial | USB | MSP
created: 2026-05-14
updated: 2026-05-14
tags: [protocol, reverse, cli, configurator, stub]
---

# CLI Internals

## Overview
<!-- How Betaflight's CLI parses commands, dispatches handlers, and serializes settings. Tokenizer, command table, `get`/`set`/`dump` semantics, `save` flow (writes EEPROM, reboots). -->

> [!gap]
> Stub. Pending: tokenizer walk, command table layout, parameter-group (`pg_`) coupling, MSP transport bridge.

## Related
- [[MSP Protocol]] (CLI commands also reachable via MSP)
- [[EEPROM Layout]]

## Source anchors
- `src/main/cli/cli.c`, `src/main/cli/settings.c`, `src/main/cli/cli_diff.c`

## Sources
-
