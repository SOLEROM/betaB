---
type: source
source_type: blog
title: "Glitching STM32 Read-Out Protection with Voltage Fault Injection"
author: "Anvil Secure"
url: https://www.anvilsecure.com/blog/glitching-stm32-read-out-protection-with-voltage-fault-injection.html
confidence: high
fetched: 2026-05-12
tags: [source, hardware-attack, rdp, glitching]
---

# Anvil Secure — VFI Against STM32 RDP

## Key claims
- **RDP levels**:
  - **Level 0** — factory default, no protection.
  - **Level 1** — flash, backup SRAM, registers blocked except when booted from user flash. Debug attempts cause a hard fault. *Reversible* (back to Level 0 erases flash).
  - **Level 2** — Level 1 plus SWD/JTAG permanently fused off; option bytes locked. *Irreversible.*
- **Target**: STM32F401CC.
- **Attack vector**: bootloader's conditional branch at `0x40023C14` that checks RDP status before serving a memory-read command. VFI skips the branch and the read goes through.
- **Timing**: command issued at 182.3 µs into bootloader execution; glitch offset 18,230–18,370 clock cycles from a reference edge; 10 ns clock period.
- **Injection points**: `VCAP_1` and `VCAP_2` pins — these are direct connections to the CPU core rail.
- **Equipment**: ChipWhisperer-Lite (fault controller), USB-UART (bootloader comms), logic analyser, modded BOOT0 pin to force system-bootloader execution on every reset.
- **Hardware mods required**: desolder bypass capacitors so the glitch isn't absorbed.

## Wider context (cross-ref)
- **CTXz `stm32f1-picopwner`** — Raspberry Pi Pico re-implementation of the Obermaier / Schink / Moczek FPB-exception glitch against STM32F1 RDP-1.
- **Joe Grand 2022** — recovered $2 M in lost crypto by glitching a Trezor One (STM32F2), building on Kraken Security Labs research.

## Why it matters
Even a firmware that is "protected" against extraction is *not* protected against a determined attacker with a $200 ChipWhisperer. Once flash is dumped, the [[Cortex-M Binary Patching]] pipeline takes over.

## Confidence
**high** — security firm publication with reproducible timing data and named target chip.
