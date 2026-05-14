---
type: index
status: stable
created: 2026-05-14
updated: 2026-05-14
tags: [architecture, source, betaflight, master, srclatest]
source-commit: 6434dd725
source-branch: master
source-version: 4.5.0-1097-g6434dd725
---

# srclatest — Betaflight Master Source Code Walkthrough

This folder is a top-down architectural map of **`/mnt/betab/betaflight/`** — the upstream Betaflight master submodule, pinned at commit `6434dd725` (`4.5.0-1097-g6434dd725`, master branch ahead of the 4.5 release).

> Purpose: read these pages once and know how the firmware is built well enough to **change** it — add a setting, port a sensor, support a new RX protocol, ship a custom build.

If you also want the 4.5-maintenance checkout (used for GUI work with Configurator 10.10.0), see `betaflight_4.5/`. The architecture below is identical in shape; only specific files (new sensors, version bumps, new MSP opcodes) differ.

## Reading order

| # | Page | What you learn |
|---|------|----------------|
| 01 | [[01-overview]] | The 5-layer architecture, gyro-driven heartbeat, PLATFORM/TARGET/CONFIG triaxis. **Start here.** |
| 02 | [[02-directory-layout]] | Every directory in `src/main/`, `src/platform/`, `lib/main/` and what lives there. |
| 03 | [[03-build-system]] | `make TARGET=X` flow, `mk/*.mk` roles, source aggregation, toolchain, outputs. |
| 04 | [[04-boot-and-scheduler]] | `main()` → 3-phase init → cooperative scheduler, REALTIME tasks, gyro tick. |
| 05 | [[05-flight-core-loop]] | `taskMainPidLoop` subtask chain: RC → PID → mixer → motors. Modes, arming. |
| 06 | [[06-flight-modules]] | Inventory of `flight/` and `fc/` — every file's role in one line. |
| 07 | [[07-hal-and-drivers]] | The HAL split: abstract `drivers/` ↔ platform-specific `src/platform/*/`. Sensor discovery. |
| 08 | [[08-io-subsystems]] | Application-level I/O: serial dispatcher, VTX, GPS, LED strip, beeper, OSD plumbing. |
| 09 | [[09-msp-cli-cms]] | The three configuration interfaces — wire protocol, terminal, on-screen menu. |
| 10 | [[10-osd-blackbox-telemetry]] | On-screen display elements, flight data logger, downlink telemetry protocols. |
| 11 | [[11-rx-subsystem]] | Receiver protocols — serial (SBUS/CRSF/GHST/iBus/SRXL2) and SPI (ELRS/CC2500/CYRF/A7105/NRF24). |
| 12 | [[12-config-and-pg]] | Parameter Groups: how settings are versioned, EEPROM-backed, hash-tracked. |
| 13 | [[13-modification-guide]] | Practical cookbook — "where do I edit to change X?" reverse-pointer. |

## Quick cross-links

- Protocol details for the wire side: [[MSP Protocol]], [[CRSF Protocol]], [[MSP Commands Reference]]
- Build system from the OS side: see `sitl/README.md` and `hw/formats.md` in the repo root
- Reverse-engineering output formats: [[Blackbox Format]], [[OSD Font Format]], [[EEPROM Layout]]

## Conventions on these pages

- File paths are absolute from repo root, e.g. `src/main/fc/core.c:1405`.
- Line numbers reflect commit `6434dd725`. Drift if the upstream moves.
- Where a page restates info that already lives elsewhere in the wiki, the existing page is linked with `[[...]]` rather than duplicated.
- "REALTIME" capitalised = `TASK_PRIORITY_REALTIME`. "FAST_CODE" capitalised = the linker section attribute used to keep hot functions in tightly-coupled memory.
