---
noteId: "d20792004f9c11f194a2c3b1eecd91b7"
type: protocol
title: "Bootloader"
status: stub
direction: host-to-fc
transport: USB DFU | UART
created: 2026-05-14
updated: 2026-05-14
tags: [protocol, reverse, bootloader, dfu, stub]
---

# Bootloader

## Overview
<!-- STM32 system bootloader (ROM DFU) + Betaflight bootloader handoff. How [[MSP Protocol]] `MSP_REBOOT(68)` with payload `1` jumps to DFU; how firmware is flashed. -->

> [!gap]
> Stub. Pending: ROM bootloader entry, BOOT0 pin handling, BF-side reboot-to-bootloader path, dfu-util workflow.

## Related
- [[MSP Protocol]] — `MSP_REBOOT` variants (0=firmware, 1=DFU, 2=MSC, 3=MSC+UTC)
- [[Cortex-M Firmware Dumping]]
- [[Bin to Hex Conversion and Constant Patching]]

## Implementation in BF
- `src/main/fc/fc_init.c`, `src/main/drivers/system_stm32*.c` (`systemResetToBootloader`)

## Sources
-
