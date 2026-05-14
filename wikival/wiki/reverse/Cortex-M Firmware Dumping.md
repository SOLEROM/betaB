---
type: protocol
title: "Cortex-M Firmware Dumping"
status: documented
direction: fc-to-host
transport: SWD/JTAG/USB/UART
created: 2026-05-12
updated: 2026-05-12
tags: [protocol, reverse, dumping]
---

# Cortex-M Firmware Dumping

## Overview
Acquiring the contents of internal flash from a Cortex-M device. The hard cases — locked devices — are handled in [[Readout Protection (STM32 RDP)]]. This page focuses on the *unlocked* paths, which cover ~90 % of consumer drone hardware.

## The Four Paths

### 1. SWD via OpenOCD + ST-Link
The default for STM32. ST-Link is $5 on AliExpress; OpenOCD is open source.

```bash
# 1. Connect SWDIO, SWCLK, GND, (optionally NRST) to the target.
# 2. Read the entire 512 KB flash to firmware.bin:
openocd -f interface/stlink-v2.cfg \
        -f target/stm32f7x.cfg \
        -c "init" \
        -c "reset halt" \
        -c "flash read_bank 0 firmware.bin 0 0x80000" \
        -c "exit"
```

(Source: [[techmaker-stm32-re]])

### 2. SWD via st-flash (libstlink)
Smaller, faster than OpenOCD, no TCL:

```bash
st-flash read firmware.bin 0x08000000 0x80000
```

### 3. USB DFU via dfu-util
The system bootloader (built into ROM, can't be erased) supports DFU on most STM32s. Enter DFU by pulling BOOT0 high and resetting:

```bash
dfu-util -a 0 -s 0x08000000:524288 -U firmware.bin
```

(Works without any debugger hardware — only a USB cable.)

### 4. UART bootloader
Older STM32s also speak the ST bootloader protocol on USART1. STM32CubeProgrammer or `stm32flash` is the client:

```bash
stm32flash -r firmware.bin /dev/ttyUSB0
```

## What You Get
Either a flat `.bin` (length = flash size) or an Intel-HEX `.hex`. The bin is what every disassembler ([[Loading Cortex-M Firmware in Ghidra]], `radare2`, IDA) wants.

## Triaging the Dump

```bash
strings firmware.bin | head -100         # human-readable text
binwalk firmware.bin                     # signatures, embedded blobs
xxd firmware.bin | head -20              # vector table by eye
entropy -v firmware.bin                  # encryption / compression hint
```

The first 8 bytes are the vector table head:

```
00000000: 00 00 04 20  ad 01 00 08
          └──MSP────┘  └──Reset──┘   (little-endian)
                                    MSP = 0x20040000  (top of 256 KB SRAM)
                                    Reset = 0x080001ad  (LSB set = Thumb)
```

Already we know: SRAM is 256 KB at `0x20000000`, the entry point is `0x080001ac` (LSB cleared), the linker put it just past the table. This is enough to point [[Ghidra]] at the right address.

## Locked Targets
If `flash read_bank` returns all `0xFF` or an "RDP active" error, the device is at Level 1 or 2. See [[Readout Protection (STM32 RDP)]] and [[Voltage Fault Injection]] for the bypass paths.

## Implementation in BF
Betaflight has no countermeasure against dumping — RDP is left at Level 0 in the upstream codebase. Vendors who ship locked retail boards do so by writing the RDP option byte before shipment.

## Gaps
> [!gap]
> Concrete BF-specific recipe: dumping an F7 board with the Betaflight bootloader still active (PA10 boot pad pulled high) vs. forcing the ROM bootloader. Worth a follow-up.

## Sources
- [[techmaker-stm32-re]]
- [[anvil-glitching-stm32-rdp]]
- [[cjacker-opensource-toolchain-stm32]]
