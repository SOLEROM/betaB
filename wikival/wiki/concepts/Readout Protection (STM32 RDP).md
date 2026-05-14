---
type: concept
title: "Readout Protection (STM32 RDP)"
status: developing
domain: security
created: 2026-05-12
updated: 2026-05-12
tags: [concept, security, stm32, anti-reversing]
---

# Readout Protection (STM32 RDP)

## Definition
A flash-protection mechanism in every modern STM32 that gates external access (SWD/JTAG, bootloader, DMA) to internal flash and backup SRAM. Configured via an **option byte** (`RDP`) stored in the flash option area; the value is read at every reset.

## Levels

| Level | Value (typical) | What's blocked | Reversible? |
|-------|-----------------|---------------|------------|
| 0 | `0xAA` | Nothing — factory default | n/a |
| 1 | any value ≠ `0xAA` and ≠ `0xCC` | Flash + backup SRAM unreadable from outside; debug attempts hard-fault | Yes — going back to 0 *mass-erases* flash |
| 2 | `0xCC` | Level 1 *plus* SWD/JTAG permanently fused off; option bytes locked | **No** — one-way fuse |

(Source: [[anvil-glitching-stm32-rdp]])

## Theory of Operation
On reset, the boot ROM reads the option byte. If RDP ≠ 0, the bootloader's `read_memory` / `read_flash` commands branch to a "permission denied" path before serving the response. SWD/JTAG silicon-level gates clamp debug-bus reads when RDP ≥ 1.

The transition Level 1 → Level 0 triggers a *mandatory mass erase* in hardware before lifting the gate, so an attacker can downgrade but not recover the contents.

## In Betaflight
Betaflight ships RDP at **Level 0** (the developer-friendly default). Vendors who customise Betaflight for retail boards usually leave it at 0 too; some lock down at Level 1 to deter copy-cats. Level 2 is rare on FPV gear because it would prevent legitimate firmware updates via SWD.

## Bypass Techniques (when source isn't available)

### Voltage Fault Injection (Level 1 → dump)
- Brown out the core during the RDP check; the CPU "skips" the conditional branch.
- Specific to the bootloader implementation — the Anvil writeup targets a check at `0x40023C14` on STM32F401CC (Source: [[anvil-glitching-stm32-rdp]]).
- Equipment: ChipWhisperer or Pi Pico glitcher; desolder bypass caps; tap VCAP_1/VCAP_2 rail.
- See [[Voltage Fault Injection]] in this wiki and the CTXz `stm32f1-picopwner` project for an RDP-1 break on STM32F1.

### Exception-based downgrade (Obermaier/Schink/Moczek)
- On STM32F1 the Flash Patch & Breakpoint unit can be configured *while* RDP-1 is active; certain remapped exception fetches end up reading bytes that RDP should have blocked. No glitching required for the F1 variant — pure software bug.

### Clock / EM glitching
- Same principle as VFI but coupling via clock line or EM probe instead of power.

### Level 2 is, in practice, unbroken
- No public attack downgrades Level 2 to readable state.
- Counter-example: Joe Grand 2022 recovered $2 M of crypto from a Trezor One whose firmware was RDP-1 (not 2); VFI worked.

## Key Relationships
- Sits on top of: [[Vector Table]] (the RDP check is *part of* the boot path).
- Bypassed by: [[Voltage Fault Injection]], FPB-exception glitch.
- Once bypassed, feeds: [[Cortex-M Firmware Dumping]] → [[Loading Cortex-M Firmware in Ghidra]] → [[Cortex-M Binary Patching]].

> [!key-insight]
> RDP is a speed bump, not a wall, unless you go to Level 2 — and even then, only because the attack surface (debug pins) is *physically* disabled. Treat any Level-1 device as "readable by a $200 attacker."

## Sources
- [[anvil-glitching-stm32-rdp]]
