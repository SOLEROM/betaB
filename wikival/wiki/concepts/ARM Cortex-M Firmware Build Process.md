---
type: concept
title: "ARM Cortex-M Firmware Build Process"
status: developing
domain: build-system
created: 2026-05-12
updated: 2026-05-12
tags: [concept, build-system, cortex-m, toolchain]
---

# ARM Cortex-M Firmware Build Process

## Definition
The pipeline that turns C / C++ / assembly sources into a flat binary the bootloader can copy into flash. For Betaflight the inputs are `src/main/**/*.c`, plus CMSIS headers, plus an MCU-family linker script; the output is `betaflight_<ver>_<TARGET>.hex`.

## The Stages

```
.c / .S    ──►  cpp (preprocess)
              │
              ▼
       arm-none-eabi-gcc -c   ──►  .o  (relocatable ELF, addresses unresolved)
                                │
                                ▼
                      arm-none-eabi-ld           (linker script supplies the memory map)
                                │
                                ▼
                          firmware.elf  (fully linked, symbol-rich)
                                │
                                ▼
                      arm-none-eabi-objcopy -O ihex   →  firmware.hex
                      arm-none-eabi-objcopy -O binary →  firmware.bin
                      arm-none-eabi-size              →  flash/RAM usage report
                                │
                                ▼
                          flash via SWD / JTAG / DFU
```

(Source: [[memfault-linker-scripts]], [[aticleworld-stm32-build]], [[cjacker-opensource-toolchain-stm32]])

## Stage Details

### 1. Preprocess
`#include` expansion, macro substitution, `#ifdef` selection. For Betaflight this is where **build options** (`USE_GPS`, `USE_OSD_SD`, etc.) gate entire features in or out (Source: [[Cloud Build System]]).

### 2. Compile
Per-file translation to Thumb-2 machine code. Output is a relocatable ELF (`.o`) where every external reference is a placeholder. The compiler is *bare-metal*: no OS calls, no `printf` that talks to a console, no libc memory allocator unless the project provides one.

### 3. Link
`arm-none-eabi-ld` reads the **linker script** (one per MCU family: e.g. `STM32F722.ld`) and:
- Assigns each section (`.text`, `.rodata`, `.data`, `.bss`) to a memory region.
- Resolves every cross-file symbol.
- Emits the **vector table** at flash offset 0 (Source: [[memfault-linker-scripts]]).
- Generates helper symbols (`_sdata`, `_edata`, `_sbss`, `_ebss`) for the reset handler.

### 4. Object copy
`objcopy` strips the ELF metadata and produces the flat image the programmer expects:
- `.bin` — raw flash bytes, base address implicit.
- `.hex` — Intel HEX text format, base address embedded; what Betaflight Configurator flashes.

### 5. Flash
Three paths — see [[Cortex-M Firmware Dumping]] for the inverse direction:
- **SWD / JTAG** via [[Ghidra]] / OpenOCD / ST-Link Utility / J-Link.
- **USB DFU** via `dfu-util` or STM32CubeProgrammer (uses the system bootloader baked into ROM).
- **Vendor bootloader over UART** (Betaflight installs a custom MSP bootloader as a fallback).

## The Linker Script (closer look)

```
MEMORY {
  FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 512K
  RAM   (rwx): ORIGIN = 0x20000000, LENGTH = 256K
}
SECTIONS {
  .text : { KEEP(*(.vectors))  *(.text*)  } > FLASH
  .data : AT(LOADADDR(.text) + SIZEOF(.text)) { *(.data*) } > RAM
  .bss  (NOLOAD) : { *(.bss*) } > RAM
}
```

Key points (Source: [[memfault-linker-scripts]]):
- `KEEP(*(.vectors))` pins the vector table to flash offset 0 so the **MSP** (initial stack pointer) sits at `0x08000000` and the **Reset_Handler** address at `0x08000004`.
- `.data` has two addresses: **LMA** (load address, in flash) and **VMA** (virtual address, in RAM). The reset handler copies LMA→VMA before `main()` runs.
- `.bss (NOLOAD)` reserves RAM without occupying space in the `.bin` file.

## After Reset (what runs before main)

1. CPU loads MSP from `0x08000000`.
2. CPU loads PC from `0x08000004` → jumps to `Reset_Handler`.
3. `Reset_Handler`:
   - copies `.data` from LMA to VMA;
   - zeroes `.bss`;
   - configures clocks (HSE / PLL);
   - calls `SystemInit()`;
   - calls `main()`.

## In Betaflight
- `src/main/startup/startup_stm32f7xx.s` is the BF reset handler.
- `make TARGET=STM32F7X2 -j` invokes the pipeline above.
- Cloud Build performs steps 1–4 server-side; the user only downloads the `.hex` (Source: [[Cloud Build System]]).

## Key Relationships
- Built **with**: [[arm-none-eabi-gcc]] (compiler), `arm-none-eabi-ld` (linker), `arm-none-eabi-objcopy` (formatter).
- Built **for**: [[STM32F722]] / [[STM32 MCU Family in Betaflight]].
- Placement governed by: linker script + [[Vector Table]].
- Inverse operation: [[Cortex-M Firmware Dumping]] + [[Loading Cortex-M Firmware in Ghidra]].

> [!key-insight]
> The vector table is the contract between hardware and software. Every binary the toolchain produces — and every dump a reverse engineer pulls — starts with the same 32-bit MSP + 32-bit Reset_Handler pair. That single fact lets you align an unknown image inside Ghidra without ever seeing the source.

## Sources
- [[memfault-linker-scripts]]
- [[aticleworld-stm32-build]]
- [[cjacker-opensource-toolchain-stm32]]
