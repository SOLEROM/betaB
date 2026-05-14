---
noteId: "03d4ff704f9811f194a2c3b1eecd91b7"
tags: []

---

# Dumping firmware off a real FC

Three ways to read the flash off a flashed Betaflight target, in order of
practicality. The Configurator itself **does not expose firmware readback**
— neither does the BF CLI. You bypass BF entirely and talk to the STM32
silicon directly.

---

## Why Configurator can't do it

The BF CLI's `dump` command writes a list of `set foo = bar` lines —
that's the **config**, not the firmware. Configurator can flash, read
config, and read Blackbox logs, but no MSP command exposes flash bytes.
Firmware self-readback would also be a security/integrity hazard for
boards that care about IP protection — so it just isn't wired up.

What *does* expose flash: the STM32's own factory ROM bootloader and the
SWD debug port. Both are below the BF layer.

---

## Method 1 — USB DFU + `dfu-util` (no extra hardware)

Every STM32 has a DFU bootloader in mask ROM. Configurator uses it to
*flash* the FC; the same protocol can also *upload* (DFU jargon for
"device → host").

### Steps

```bash
# 1. Boot the FC into DFU mode. Two options:
#    (a) Connect Configurator → top-right "Activate boot loader" button
#    (b) Hold the BOOT0 pad high (3.3V) while plugging USB in
#        (pad location is board-specific — check your FC's docs)

# 2. Confirm the chip is enumerated as DFU:
dfu-util -l
# Expect a line like:
#   Found DFU: [0483:df11] ... "@Internal Flash  /0x08000000/04*016Kg,01*064Kg,07*128Kg"

# 3. Dump the entire flash bank. Size is MCU-dependent:
#       F405 / F411 / F722 → 1 MB  (0x100000)
#       F745 / F765        → 1 MB
#       H743               → 2 MB  (0x200000)
dfu-util -a 0 -s 0x08000000:0x100000 -U fc_dump.bin

# 4. (Optional) Convert to ihex for diffing against a build artifact:
arm-none-eabi-objcopy -I binary -O ihex \
    --change-addresses 0x08000000 fc_dump.bin fc_dump.hex
```

`fc_dump.bin` is a flat image of the entire flash bank, including unused
0xFF-padded regions after the firmware. To compare against a build
artifact, trim it to firmware size first — see [formats.md](formats.md).

### When to use this

Default choice. No probe needed, no soldering, works as long as the FC
USB-enumerates and you can reach the BOOT0 pad (or the FC is responsive
enough to honor Configurator's "Activate boot loader" button).

---

## Method 2 — SWD via ST-Link probe (~$3 clone)

If USB DFU isn't an option — FC is bricked, BOOT0 isn't broken out,
firmware has hung — the SWD debug port is independent of CPU state. You
do need physical access to **SWDIO**, **SWCLK**, **GND** pads (often
test points on the underside).

### With OpenOCD

```bash
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg \
    -c "init; halt; flash read_bank 0 fc_dump.bin; exit"
```

Adjust the target cfg for your MCU family:
- `target/stm32f4x.cfg` — F4 family
- `target/stm32f7x.cfg` — F7 family
- `target/stm32h7x.cfg` — H7 family

### With STM32CubeProgrammer (ST's official CLI)

```bash
STM32_Programmer_CLI -c port=SWD -r32 0x08000000 0x100000 fc_dump.bin
```

### When to use this

- The FC is bricked or hung (DFU won't enumerate).
- You suspect runtime tampering (SWD reads the *current* flash state
  without booting any code).
- You want to also dump RAM, option bytes, OTP, or the system ROM.

---

## Method 3 — UART system-memory bootloader + `stm32flash`

The STM32 ROM bootloader also speaks UART. Needs BOOT0 high *and* a
USB-UART adapter wired to the FC's UART1 RX/TX. Rare on FCs because USB
DFU is just easier — included for completeness.

```bash
stm32flash -r dump.bin /dev/ttyUSB0
```

### When to use this

Niche. Useful if USB enumeration is broken at the board level (bad USB
PHY, blown VBUS protection) but the MCU itself is fine.

---

## The RDP gotcha

STM32 option bytes contain a 1-byte field called **Read Out Protection
(RDP)** that controls whether flash is readable at all.

| Level | Effect |
|-------|--------|
| **0** (default, `0xAA`) | Flash readable freely via DFU/SWD/UART. |
| **1** (any other byte except `0xCC`) | Debug disabled; attempting readout **mass-erases the flash**. You get an empty dump and a wiped FC. |
| **2** (`0xCC`) | Permanent lock. Debug fused off forever. Rare on FCs. |

**Stock Betaflight ships with RDP=0**. Vanilla BF firmware on a vanilla
FC dumps cleanly. Some commercial vendors with proprietary forks ship
their boards at RDP=1 to protect their IP — those firmwares are
unreadable without invasive techniques (chip decapping, glitching), well
outside hobbyist territory.

### Check RDP *before* attempting a dump

```bash
# Read option bytes via DFU (alt setting 1 is "Option Bytes" on STM32):
dfu-util -a 1 -s 0x1FFFC000:0x10 -U option_bytes.bin

# The RDP byte is at offset 0:
xxd option_bytes.bin | head -1
#   0xAA = level 0 (unlocked — safe to dump)
#   0xCC = level 2 (permanently locked)
#   anything else = level 1 (READING WILL ERASE — abort)
```

If RDP is non-zero, **do not attempt a dump via DFU or SWD**. The chip
will erase itself on the first read attempt. Reflash Betaflight from
your own artifact instead.

---

## What you can do with a dump

Once you have `fc_dump.bin`, you have a byte-identical view of what the
MCU is running. From there:

- **Verify integrity:** diff against your build artifact (see
  [formats.md](formats.md) for the trim/pad recipe). A clean flash with
  no bit rot, no rogue modifications, no bootloader tampering should
  match exactly.
- **Reverse-engineer:** find constants, strings, jump tables. The
  reverse-engineering skills you practiced on the SITL ELF transfer
  directly — the bin is the same kind of `.rodata` + `.text` blob, just
  ARM Thumb-2 instead of x86-64.
- **Patch and reflash:** patch bytes in the bin, convert back to hex
  (see [formats.md](formats.md)), flash through Configurator. The
  Configurator's header will show whatever you patched.

---

## Files in this directory

| File | Purpose |
|------|---------|
| `dumps.md` | This file — how to read firmware off real hardware. |
| `formats.md` | The three byte-identical views (ELF / hex / bin) and conversions. |
