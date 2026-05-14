---
type: protocol
title: "Cortex-M Binary Patching"
status: documented
direction: host-only
transport: filesystem
created: 2026-05-12
updated: 2026-05-12
tags: [protocol, reverse, patching]
---

# Cortex-M Binary Patching

## Overview
Changing the behaviour of a Cortex-M firmware image when you do not have source code. Three escalating techniques: **byte-level patches**, **veneers** (jumping out to free space), and **FPB-based live patches** (no flash modification at all).

(Source: [[giese-defcon26-cortex-m-modify]])

## When you only need to flip a constant

If the parameter you want to change is a literal stored in `.rodata` or in a `mov.w` immediate, the patch is a hex-edit:

```
1. Find the value with strings / xxd / Ghidra search-for-bytes.
2. Confirm xref leads to the right routine.
3. Overwrite the bytes in the .bin.
4. Recompute / disable the vendor checksum.
5. Reflash.
```

Examples: changing a baud-rate table, raising a hard-coded throttle limit, swapping a string.

## When you need to skip a check

The classic "always-true" patch: replace a conditional branch (`cbz`/`cbnz`/`bne`) with `b` (unconditional) or `nop` (fall through). Both are 16-bit Thumb instructions, so the byte-count stays the same — no relocation needed (Source: [[techmaker-stm32-re]]).

| Original | Bytes (LE) | Patched | Bytes (LE) | Effect |
|---|---|---|---|---|
| `BEQ +0x10` | `08 D0` | `B +0x10` | `08 E0` | always taken |
| `BNE +0x10` | `08 D1` | `NOP` | `00 BF` | never taken, fall through |

## When you need to add new code (veneers)

Step 1 — find free flash. Common locations:
- Tail of `.text` (linker often leaves it `0xFF`-filled to the next sector boundary).
- Unused IRQ vector slots in the [[Vector Table]] (every entry that points to a default-handler stub).
- Padding between functions, especially after the vendor's "config" blob.

Step 2 — write the patch in assembly (or compile a tiny `.S` with `arm-none-eabi-as`), placing it at the free address.

Step 3 — at the call site, overwrite the original `BL target` (4 bytes) with `BL veneer`. The veneer:
1. Does whatever extra work you want.
2. Tail-calls the original target.
3. Returns to the caller.

Step 4 — recompute the vendor checksum if the image carries one (many bootloaders verify a CRC32 over `.text`).

**Nexmon** is the canonical framework for this kind of patching on Cortex-M — it lets you write the veneers in C and links them against the donor binary (Source: [[giese-defcon26-cortex-m-modify]]).

## When you can't (or don't want to) modify flash: FPB

The **Flash Patch and Breakpoint** unit in Cortex-M3/M4/M7 contains 6–8 hardware comparators. Each can:
- **Remap** a flash address to a different flash or SRAM address (literal patching, code patching).
- **Trigger** a breakpoint without modifying the instruction stream.

Workflow:
1. Get a debugger or downloader onto the device after boot.
2. Write the original instruction's address into `FP_COMPn`.
3. Write the redirect target into `FP_REMAPn`.
4. Enable the unit via `FP_CTRL`.

The CPU now fetches from the redirected address whenever it encounters the original. No flash write happens, so the patch survives only until reset — but it can be re-applied on every boot by a small "loader" living in unprotected flash. Used both legitimately (silicon errata workarounds) and offensively (the Obermaier/Schink/Moczek STM32F1 RDP-1 bypass; see [[Readout Protection (STM32 RDP)]]).

## Bookkeeping

After any patch:

- **Checksum**. If the vendor verifies, find the routine (it'll xref the address ranges defining its window), then either patch the verifier out, or recompute the CRC over the new image.
- **Signature**. If the image is RSA/ECDSA-signed, byte patches will be rejected. Either find a missing-check bug (common in older STM32H7 secure boot implementations) or move the patch into a slot the verifier ignores (e.g., past the signed region).
- **Reflash**. SWD via [[Cortex-M Firmware Dumping]] in reverse, or DFU.

## Patching in the Wild
- **Cleanflight → Betaflight** (the legitimate case): full source rewrite, not a patch — but useful for understanding which functions BF cares about modifying.
- **Custom-firmware retail vacuums / cameras**: pure veneer-and-reflash. Dennis Giese DEFCON 26 talk is the reference (Source: [[giese-defcon26-cortex-m-modify]]).
- **Bricked FPV gear recovery**: hex-edit the bad config blob in a backup `.bin`, reflash. Common when a `cli` paste corrupts persistent config.

## Frameworks & Tools
- **[[Ghidra]] / IDA / radare2** — disassembly + finding bytes to patch.
- **[[SVD-Loader]]** — turns MMIO addresses into named registers.
- **Nexmon** — C-level binary patching for Cortex-M.
- **arm-none-eabi-as / ld** — assembling veneers in raw form.
- **STM32CubeProgrammer / openocd / st-flash / dfu-util** — reflash path.

## Gaps
> [!gap]
> Concrete worked example of patching a Betaflight feature gate (e.g. unlocking GPS rescue on a build that shipped without it). The Cloud Build System makes this less needed, but it's a useful teaching example.

## Sources
- [[giese-defcon26-cortex-m-modify]]
- [[techmaker-stm32-re]]
- [[wrongbaud-ghidra-stm32-loader]]
