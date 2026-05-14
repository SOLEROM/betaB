---
noteId: "2446a9704f9811f194a2c3b1eecd91b7"
tags: []

---

# The three byte-identical views: ELF / hex / bin

A Betaflight cross-build produces three files in `obj/` that all describe
the same firmware image at the byte level. They differ only in
*packaging*: which metadata is present, what's human-readable, what's
flashable. Knowing which to reach for is most of the workflow.

The walkthrough below uses a real artifact from our cross-build:
`betaflight_2026.6.0-alpha_STM32F405`.

---

## At a glance

| View | Size | What it has | Reach for it when… |
|------|------|-------------|---------------------|
| **`.elf`** | 6–10 MB | Symbols, debug info, section table, the firmware bytes themselves | You need to find *what code/data sits at what address*. Disassembly, symbol lookup, mapping a `0x080xxxxx` address to a function name. |
| **`.hex`** | 1.5–2 MB | Firmware bytes + addresses, all ASCII | Flashing via Configurator / DFU / ST tools. Human-readable inspection. |
| **`.bin`** | 600–800 KB | Raw firmware bytes, no metadata | Patching, diffing, comparing against `dfu-util` dumps. |

Roughly: **ELF is for understanding, hex is for flashing, bin is for
patching.**

---

## The view in detail

### `.elf` — Executable and Linkable Format

The ELF is the cross-compiler's primary output. It's a structured
container with multiple sections (`.text`, `.rodata`, `.data`, `.bss`,
`.debug_*`, symbol tables, etc.), each tagged with the **virtual address
it'll occupy at runtime**.

```
ELF header
├─ .text          (code) at 0x08008000, 380 KB
├─ .rodata        (const data, strings) at 0x080xxxxx, 90 KB
├─ .data          (initialized RW) at 0x20000000, 8 KB
├─ .bss           (zeroed RW, no bytes on flash) at 0x20002000, 60 KB
├─ .symtab        (symbol → address table)
├─ .debug_info    (DWARF debug info — function names, line numbers)
└─ ...
```

This is what `objdump`, `readelf`, `nm`, `addr2line`, and GDB consume.
Nothing here gets flashed directly — the flashing tools strip away
everything except `.text` + `.rodata` + `.data` (the "load segments").

**Why it's big:** debug info is huge. Stripping with `arm-none-eabi-strip
-g` typically cuts the ELF to ~700 KB.

### `.hex` — Intel HEX

A line-oriented ASCII format. Every record carries:

```
:10082000 1F E5 0A C0 1B EA 0B 80 4F F4 80 30 02 21 03 4A  XX
 │  │  │  └────────────── 16 bytes of payload ──────────┘  └─ checksum
 │  │  └─ address (relative; high bits set by separate "extended address" records)
 │  └─ record type (00 = data)
 └─ byte count
```

So `.hex` carries **bytes + addresses + checksums** in plain ASCII. About
**2.5–3× the size of the raw bytes** — each 16-byte payload becomes ~45
ASCII chars.

**Why it exists:** dates back to PROM programmers in the 1970s. ASCII
made it transmittable over serial without binary-safe links. Modern
flashing tools (Configurator, `dfu-util`, ST-Link Utility,
`stm32flash`) all accept it because the embedded ecosystem has carried
it forward.

### `.bin` — flat binary

Just the raw bytes. No headers, no addresses, no metadata. **Offset 0 in
the file corresponds to the firmware's lowest flash address** (typically
`0x08000000` for STM32).

This is what the MCU's flash actually contains, byte-for-byte. It's also
what `dfu-util -U` produces when it reads flash back.

**The metadata loss matters when you go back the other way** — see the
`--change-addresses` flag below.

---

## Conversions: `objcopy` does all of them

`arm-none-eabi-objcopy` (and plain host `objcopy` for the SITL case) is
the swiss-army-knife. It reads any format it knows, writes any format
it knows. The `-I` / `-O` flags pick input and output:

```
                    objcopy -I ihex -O binary
        ┌──────────────────────────────────────────┐
        │                                          ▼
     ┌──────┐  -O ihex   ┌──────┐  -O binary    ┌──────┐
     │ .elf │ ─────────► │ .hex │ ─────────────►│ .bin │
     └──────┘            └──────┘               └──────┘
        ▲                   ▲                      │
        │                   │  -I binary           │
        │                   │  -O ihex             │
        │                   │  --change-addresses  │
        │                   └──────────────────────┘
        │                                          │
        └─ no path back (debug info is gone) ──────┘
```

### ELF → hex (during build, but you can redo it)

```bash
arm-none-eabi-objcopy -O ihex \
    betaflight_2026.6.0-alpha_STM32F405.elf \
    betaflight_2026.6.0-alpha_STM32F405.hex
```

The build pipeline already does this; you'd only redo it manually if
you'd modified the ELF (e.g. patched a constant via `objcopy
--update-section`).

### ELF → bin

```bash
arm-none-eabi-objcopy -O binary \
    betaflight_2026.6.0-alpha_STM32F405.elf \
    betaflight_2026.6.0-alpha_STM32F405.bin
```

### hex → bin (most common — for analysis/diffing)

```bash
objcopy -I ihex -O binary \
    betaflight_2026.6.0-alpha_STM32F405.hex \
    betaflight_2026.6.0-alpha_STM32F405.bin
```

No ARM toolchain needed — host `objcopy` handles Intel HEX. The output
size will drop dramatically (1.7 MB hex → 648 KB bin in our test):
that's the ASCII overhead disappearing.

### bin → hex (for reflashing patched bytes)

```bash
objcopy -I binary -O ihex \
    --change-addresses 0x08000000 \
    patched.bin patched.hex
```

⚠ **The `--change-addresses 0x08000000` is essential.** Flat `.bin` has
no address metadata — `objcopy` would otherwise mark it as loading at
`0x00000000`, and the flasher would happily write your patched firmware
into a region that's either non-existent or part of the bootloader. Use
the flash base for your target family:

| Family | Flash base |
|--------|------------|
| F4 / F7 / G4 | `0x08000000` |
| H7 (bank 1)  | `0x08000000` |
| H7 (bank 2)  | `0x08100000` |

### Round-trip sanity check

```bash
# After patching a bin and converting back to hex, prove the round-trip
# preserved every byte:
objcopy -I ihex -O binary patched.hex roundtrip.bin
cmp patched.bin roundtrip.bin       # silent = identical
```

---

## How they line up with a `dfu-util` dump

Both the `.bin` (from `objcopy -O binary`) and a `dfu-util -U` dump use
the **same address convention**: file offset 0 = flash base. So they're
byte-for-byte comparable — with two caveats:

| | hex-derived `.bin` | `dfu-util` dump |
|---|---|---|
| Size | exactly the firmware size | whatever range you asked for |
| Tail | ends at last written byte | padded with `0xFF` for unused flash |
| Address-0 byte | corresponds to `0x08000000` | corresponds to `0x08000000` |

To diff a hardware dump against your build, trim the dump first:

```bash
# Trim the dfu dump down to the build's exact firmware size:
FW_SIZE=$(stat -c%s betaflight_2026.6.0-alpha_STM32F405.bin)
head -c $FW_SIZE fc_dump.bin > fc_dump_trim.bin

# Now compare:
cmp betaflight_2026.6.0-alpha_STM32F405.bin fc_dump_trim.bin
# Silent → clean flash, no tampering, no bit rot.
# Position printed → that's the first divergent byte.
```

---

## Picking the right view for the job

**Finding *where* something lives (function, constant, string):**
Use the **ELF**.
```bash
arm-none-eabi-nm betaflight_..._STM32F405.elf | grep mspProcessOutCommand
arm-none-eabi-objdump -d --disassemble=mspFcProcessCommand <elf>
arm-none-eabi-strings <elf> | grep '^Betaflight'
```

**Inspecting human-readably without converting:**
Use the **hex**. Open in a text editor — you'll see addresses and bytes
side by side. Useful for spot-checks ("did this build include the
expected sections?").

**Patching bytes:**
Use the **bin**. Single contiguous file, offsets line up cleanly,
trivial to script. `dd if=patch_bytes of=fw.bin bs=1 seek=<offset>
conv=notrunc`.

**Flashing through Configurator / DFU / ST tools:**
Use the **hex**. Every flashing tool in the BF ecosystem speaks Intel
HEX natively. (Some also accept `.bin` with an explicit base address;
sticking to hex avoids the foot-gun.)

**Comparing what's on the FC to what you built:**
Convert hex → bin (or use the existing bin if you have one), dump the
FC, trim, `cmp`. Any divergence is interesting.

---

## Worked example: locating the version string in three views

The `2026.6.0-alpha` string we saw on the Configurator header lives at
the same address in all three views — just expressed differently.

```bash
# 1. Find it in the ELF — get the symbol + section:
arm-none-eabi-strings -a -t x betaflight_..._STM32F405.elf | grep '2026.6.0-alpha'
# Output: <hex offset>  2026.6.0-alpha
# Then map the offset to an address with `readelf -S <elf>` to find which section.

# 2. Find it in the hex — grep the ASCII directly:
grep -n '2026' betaflight_..._STM32F405.hex   # finds the record(s) carrying those bytes

# 3. Find it in the bin — byte offset from flash base:
grep -abo '2026\.6\.0-alpha' betaflight_..._STM32F405.bin
# Output: <byte offset>:2026.6.0-alpha
# Flash address = 0x08000000 + <byte offset>
```

The same string, three angles on the same firmware. Once you internalize
this it stops feeling like three different things — it's one image,
viewed through three different lenses.

---

## Files in this directory

| File | Purpose |
|------|---------|
| `dumps.md` | How to read firmware off real hardware (USB DFU / SWD / UART). |
| `formats.md` | This file — the ELF / hex / bin trinity and conversions between them. |
