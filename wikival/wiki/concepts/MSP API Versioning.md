---
type: concept
title: "MSP API Versioning"
status: documented
created: 2026-05-14
updated: 2026-05-14
tags: [concept, msp, versioning, handshake, configurator]
related:
  - "[[MSP Protocol]]"
  - "[[Betaflight Configurator]]"
  - "[[MSP v2 Frame Format]]"
---

# MSP API Versioning

## Overview

Every change to the MSP wire surface — a new command, a removed command, a
changed payload — must bump the **MSP API version**. Clients query the
version on connect and gate feature usage on it. This is how Betaflight
Configurator can talk to a fleet of FCs running anything from 4.3 to 5.x
without separate codepaths.

(Source: [[betaflight-msp-protocol-h]], [[betaflight-deepwiki-msp]])

## The Handshake

Standard client connect sequence (used by BF Configurator and `@betaflight/msp`):

```
1. Client → FC : MSP_API_VERSION  (cmd 1)
   FC → Client : 3 bytes
                 [0] MSP_PROTOCOL_VERSION  (currently 0)
                 [1] API_VERSION_MAJOR      (currently 1)
                 [2] API_VERSION_MINOR      (e.g. 47 → "API 1.47")

2. Client → FC : MSP_FC_VARIANT   (cmd 2)
   FC → Client : 4-byte ASCII identifier ("BTFL", "INAV", "CLFL", "BAFL")

3. Client → FC : MSP_FC_VERSION   (cmd 3)
   FC → Client : 3 bytes [major, minor, patch]

4. Client → FC : MSP_BUILD_INFO   (cmd 5)
   FC → Client : 11-byte date + 8-byte time + git short-hash
                 (used to show "compiled YYYY-MM-DD HH:MM, git abc1234")

5. Client → FC : MSP_BOARD_INFO   (cmd 4)
   FC → Client : board identifier (4 chars) + hw revision + …
```

(Source: [[betaflight-deepwiki-msp]])

## Why MSP_API_VERSION Comes First

A client that sends MSP_API_VERSION first and gets:

- A clean response → the FC speaks Cleanflight-family MSP.
- No response → legacy MultiWii. Fall back to `MSP_IDENT` (V1 command).
- Response with major version the client doesn't understand → abort
  rather than risk misparsing.

The major version is the breaking-change axis. The minor version increments
with every release that adds or alters MSP commands.

## Version Gating in Source

The Betaflight tree gates code paths with explicit version comparisons:

```c
#if API_VERSION >= 1.46
    // telemetry auto-enable logic
#endif
```

Clients do the same in JavaScript:

```js
if (semver.gte(apiVersion, '1.47.0')) {
    sendCmd(MSP_SOFTSERIAL_CONFIG);
}
```

API 1.46 introduced telemetry auto-enable; 1.47 brought MSP-gated
softserial dependencies. Each Betaflight release bumps `API_VERSION_MINOR`
in `src/main/build/version.h`.

(Source: [[betaflight-msp-protocol-h]])

## Persistence Flow

A typical Configurator "Save" click:

```
MSP_SET_FEATURE_CONFIG  → runtime config updated, not persisted
MSP_SET_PID             → runtime config updated, not persisted
...
MSP_EEPROM_WRITE (250)  → calls writeEEPROM() then readEEPROM()
                          (aborted if FC is armed)
MSP_REBOOT (68)         → motorShutdown() + systemReset()
                          (also aborted if armed)
```

The pattern is *write-many, commit-once*. EEPROM_WRITE is the commit; only
after it does the configuration survive a power cycle.

(Source: [[betaflight-deepwiki-msp]])

## MSP_REBOOT Variants

`MSP_REBOOT` accepts a 1-byte payload selecting the reboot target:

| Value | Target |
|-------|--------|
| 0 | Firmware (normal reboot via `systemReset()`) |
| 1 | Bootloader (ROM DFU entry) |
| 2 | MSC (Mass Storage Class — exposes SD card to USB host) |
| 3 | MSC with UTC time |

All variants abort if the FC is armed.

## Sources

- [[betaflight-deepwiki-msp]]
- [[betaflight-msp-protocol-h]]
- [[inav-wiki-msp-v2]]
