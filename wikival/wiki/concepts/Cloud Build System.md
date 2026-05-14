---
type: concept
title: "Cloud Build System"
status: developing
created: 2026-05-12
updated: 2026-05-12
tags: [concept, build-system, cloud-build, configurator, flash-size, f722, f411]
---

# Cloud Build System

## What It Is

The **Cloud Build System** is a feature introduced in **Betaflight 4.4**
that lets users **server-side compile a custom firmware binary** with
exactly the drivers and features they want — instead of flashing a
one-size-fits-all monolithic binary. (Source: [[Oscar Liang Betaflight
4.4]], confidence: **high**.)

The user ticks options in the Configurator, the request goes to
Betaflight's build server, and the resulting `.hex` is downloaded back
and flashed.

## Why It Exists

Two MCUs in the active Betaflight lineup have only **512 KB of flash**:

- [[STM32F722]] (specifically the F722R**E**T6 variant)
- STM32F411

As Betaflight added features release-by-release (Cloud Build itself, GPS
Rescue improvements, new gyros, RACE PRO modes, OSD enhancements, new
ESC protocols, …) the monolithic build crept toward those 512 KB
ceilings. By BF 4.4, fitting "everything" into 512 KB was no longer
viable.

The Cloud Build solution: ship one MCU-family binary that is **smaller
than the monolithic build** because the user has explicitly disabled
features they don't use. (Source: [[Oscar Liang Betaflight 4.4]],
confidence: **high**.)

## How It Works

The user-facing flow:

1. Open the Betaflight Configurator → Firmware Flasher tab.
2. Select MCU / board.
3. Click "Load Firmware Online" (Cloud Build) instead of "Load Local".
4. Choose features: gyro drivers, OSD types, ESC protocols, barometers,
   flash chips, GPS support, magnetometers, race-mode pack, etc.
5. (Optional) Enable Expert Mode to add custom defines.
6. Submit — the build server compiles, signs, and returns the binary.
7. Configurator flashes the result.

(Source: [[Oscar Liang Betaflight 4.4]], confidence: **high**.)

> [!info] Critical Constraint
> *"Unless you have selected the option for your firmware it will not be
> available once you flash the board."*
> — i.e. features you didn't tick are not present in the resulting
> binary at all. To add one later, request a new build.

## Server-Side Compile, Not Client-Side

The build runs on Betaflight's infrastructure. The Configurator is a
thin client: it submits a build request (MCU + feature list) and
downloads the artifact. (Source: [[Oscar Liang Betaflight 4.4]],
confidence: **high**.) This contrasts with traditional embedded
workflows where the user installs the toolchain and compiles locally.

## Relationship to Unified Targets

The Cloud Build sits **on top of** the unified-target system (see
[[Betaflight MCU Targets]]). Unified targets give you one binary per
MCU; Cloud Build customizes which features go into that binary.

For boards with comfortable flash budgets (H7 with 2 MB, F765 with 2 MB,
F745 with 1 MB), Cloud Build is **optional convenience**. For F411 and
F722RE, it is increasingly **necessary** to fit modern feature sets.

## Implications for the MCU Conversation

- It **extends the useful life** of 512 KB MCUs by 1–2 release cycles
  beyond what they could otherwise carry. (Source: [[Oscar Liang
  Betaflight 4.4]], confidence: **high**.)
- It **does not change the motor-output cap** on F4/F7 (which is a
  policy decision about timer/DMA capability, not flash). (Source:
  [[Betaflight Manufacturer Design Guidelines]], confidence: **high**.)
- It **does not eliminate** the eventual need for hardware upgrades; it
  delays it.

## Open Questions

> [!gap] Build server architecture
> Where does the build actually run (BF infrastructure vs GitHub Actions
> vs third-party)? Is the build reproducible?

> [!gap] Compiler flag overrides at request time
> The community-reported `OPTIMISE_SPEED → OPTIMISE_SIZE` override for
> 512 KB targets — is that always applied for cloud builds targeting
> F722RE, or only when the request would otherwise overflow?

> [!gap] Custom defines through Expert Mode
> What is the full grammar of custom defines a user can inject? Are
> they sandboxed?

## Related

- [[Betaflight MCU Targets]]
- [[STM32F722]]
- [[STM32 MCU Family in Betaflight]]

## Sources

- [[Oscar Liang Betaflight 4.4]]
- [[Betaflight Manufacturer Design Guidelines]]
- [[Flying Rabbit Creating BF Target]]
