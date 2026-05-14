---
type: concept
title: "Vector Table"
status: developing
domain: cortex-m
created: 2026-05-12
updated: 2026-05-12
tags: [concept, cortex-m, hardware]
---

# Vector Table

## Definition
A fixed-format array at the start of flash that tells the Cortex-M core where to jump for every CPU exception and IRQ. On reset the core reads the first two entries automatically — no software runs before this happens.

## Format (Cortex-M)

```
Offset    Content                                   Type
------    ---------------------------------------   ---------
+0x00     Initial Main Stack Pointer (MSP)          uint32_t value
+0x04     Reset_Handler                             function pointer
+0x08     NMI_Handler                               function pointer
+0x0C     HardFault_Handler                         function pointer
+0x10     MemManage_Handler                         function pointer
+0x14     BusFault_Handler                          function pointer
+0x18     UsageFault_Handler                        function pointer
...
+0x40     SysTick_Handler                           function pointer
+0x40+n*4 IRQn_Handler                              function pointers
```

(Source: [[memfault-linker-scripts]], [[wrongbaud-ghidra-stm32-loader]])

Function pointers are stored with their **LSB set to 1** to indicate Thumb mode — e.g. a handler at byte address `0x08000451` lives at instruction address `0x08000450`. Forgetting to set the Thumb bit causes an immediate UsageFault on call.

## In the Build Process
The linker script forces the vector table to flash offset 0 via:

```
.text : { KEEP(*(.vectors))   *(.text*) } > FLASH
```

`KEEP` prevents `--gc-sections` from discarding the array because no C code references it explicitly — the *hardware* references it.

## In Reverse Engineering
The vector table is the *anchor point* for any STM32 binary you did not build yourself.

Workflow (Source: [[wrongbaud-ghidra-stm32-loader]], [[feabhas-ghidra-cortex-m]]):
1. Load `.bin` with base `0x08000000`.
2. The 32 bits at offset 0 = MSP value (e.g. `0x20040000` for a 256 KB-SRAM part).
3. The 32 bits at offset 4 = Reset_Handler (clear the LSB to get the instruction address, then disassemble there).
4. Subsequent entries map 1:1 to the IRQs declared in CMSIS for that MCU family — labeling them automatically (via [[SVD-Loader]]) gives you instant names for every ISR.

## Vector Table Offset Register (VTOR)
A Cortex-M3/M4/M7 application can *relocate* the table by writing `SCB->VTOR`. Bootloaders use this:
- ROM bootloader's vector table lives at `0x00000000`.
- After hand-off to the user image at `0x08000000`, the user image writes `VTOR = 0x08000000` so its own table is active.
- Some projects use **multiple** vector tables (one in flash, one in RAM) for runtime ISR swapping.

> [!key-insight]
> Because the vector table format is fixed by the silicon, *every* Cortex-M binary you encounter is self-describing for at least the first 16 words. You always know where to start reading, even when stripped of every symbol.

## Sources
- [[memfault-linker-scripts]]
- [[wrongbaud-ghidra-stm32-loader]]
- [[feabhas-ghidra-cortex-m]]
