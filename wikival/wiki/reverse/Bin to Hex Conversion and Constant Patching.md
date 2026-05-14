---
type: protocol
title: "Bin → Hex Conversion and Constant Patching"
status: documented
direction: host-only
transport: filesystem
created: 2026-05-12
updated: 2026-05-12
tags: [protocol, reverse, patching, intel-hex, objcopy, workflow]
---

# Bin → Hex Conversion and Constant Patching

## Why This Page Exists

A dumped firmware ([[Cortex-M Firmware Dumping]]) is a **flat `.bin`** —
no addresses, no symbols. To flash it back through most STM32 tools
(STM32CubeProgrammer "Erase & Program" mode, Betaflight Configurator,
many third-party flashers) the file needs to be **Intel HEX** (`.hex`),
which carries its base address inside the file. This page covers the
conversion both ways, the format itself, and the practical workflow of
finding and patching a constant — using `CRSF_BAUDRATE` as the worked
example.

(Companion pages: [[Cortex-M Firmware Dumping]] for the dump, [[Cortex-M Binary Patching]] for general patching theory, [[Finding Defines in Firmware Binaries]] for the underlying principle, [[CRSF_BAUDRATE]] for the target value.)

## The Format Difference (Why You Convert)

| | `.bin` | `.hex` |
|---|---|---|
| Encoding | raw bytes | ASCII text |
| Base address | **implicit** — you have to *tell* the flasher | **embedded** in `:02000004...` records |
| Per-line checksum | none | yes (one byte per record) |
| Size | = flash size (e.g. 1 MB) | ~2.2× the `.bin` |
| What flashes accept it | dfu-util, openocd "flash write_image", CubeProgrammer with explicit address | CubeProgrammer "Program", BF Configurator, STM32 ROM bootloader (via stm32flash) |

Key fact: a `.bin` with no separate address argument is just bytes
hanging in the air. A `.hex` is a `.bin` + addresses + framing. They
carry the same payload.

## Intel HEX Format (Just Enough to Read One)

```
:020000040800F2     ← extended-linear-address record: high 16 bits of address = 0x0800
:10000000 00000420 AD010008 ...  D7     ← data record: 16 bytes at 0x0000 (→ 0x08000000)
:10001000 ...                    XX
...
:00000001FF        ← end-of-file record
```

Each line (`record`):
```
:LL AAAA TT [DD...] CC
 │  │    │   │       └ checksum: two's complement of sum of all bytes
 │  │    │   └ data bytes (LL of them)
 │  │    └ type: 00=data, 01=EOF, 04=extended linear address, 05=start linear address
 │  └ offset (low 16 bits of address — high 16 come from the last type-04 record)
 └ byte count of data
```

The `:020000040800` line tells the flasher *"everything that follows lands at `0x0800xxxx`"* — that's how the base address survives the round-trip.

## The Conversion (Both Directions)

From inside the cross env (`cd /mnt/betab/cross && make shell`):

### `.bin` → `.hex` — add the base address

```sh
arm-none-eabi-objcopy \
    -I binary \
    -O ihex \
    --change-addresses 0x08000000 \
    firmware.bin firmware.hex
```

Critical: `--change-addresses 0x08000000` is what stamps the STM32
flash base into the HEX. Forget it and the file says "load at address
0x00000000" — which most flashers will reject or honor literally,
bricking the board.

### `.hex` → `.bin` — strip back to raw bytes

```sh
arm-none-eabi-objcopy -I ihex -O binary firmware.hex firmware.bin
```

The reverse trip is lossless: HEX → BIN → HEX produces an identical
HEX (modulo record line-length defaults).

### Sanity-check the result

```sh
# First 8 bytes = vector table head (initial MSP + Reset_Handler)
xxd firmware.bin | head -1
# Or for the .hex, first data record carries them
head -3 firmware.hex
```

You should see something like `00 00 04 20  ad 01 00 08` — see [[Vector Table]] for the meaning.

## Worked Example: Patching `CRSF_BAUDRATE`

You've dumped an F405 BF firmware. You want to change the default
CRSF baud from 420 000 to, say, 921 600 — without source.

### Step 1: Know the byte pattern you're hunting

From [[CRSF_BAUDRATE]] we know the default is **420 000 = 0x00066900**.
In little-endian flash that's the four-byte sequence:

```
00 69 06 00
```

The compiler stored it either as a **literal pool entry** (4 bytes of
`.text` loaded with `ldr rN, [pc, #imm]`) or as a `movw/movt`
immediate pair (where the bytes are interleaved with instruction
encoding). Search for the literal pool form first — it's much easier
to find and patch.

### Step 2: Locate the bytes in the dump

```sh
# Quick search with xxd + grep:
xxd -p -c1 firmware.bin | tr -d '\n' \
    | grep -obE '00690600' \
    | awk -F: '{printf "0x%x  flash=0x%x\n", $1/2, 0x08000000 + $1/2}'

# Or a tiny python one-liner (cleaner):
python3 - <<'PY'
data = open('firmware.bin','rb').read()
needle = (420000).to_bytes(4, 'little')
print(f"Searching for {needle.hex()} ({420000})")
off = 0
hits = []
while True:
    i = data.find(needle, off)
    if i < 0: break
    print(f"  bin offset 0x{i:08x}  flash addr 0x{0x08000000+i:08x}")
    hits.append(i)
    off = i + 1
print(f"{len(hits)} hits")
PY
```

Expect **several hits**. Three categories:

| Hit type | What it is | Patchable? |
|---|---|---|
| Literal pool entry in `.text` | Aligned 4 bytes near a function | ✅ yes — flip the bytes |
| Inline `.data`/`.rodata` initialiser | Initialiser for a global | ✅ yes (changes the *initial* value loaded into RAM at boot) |
| Random collision | Some unrelated `.rodata` blob, jpeg, etc. that happens to contain `00 69 06 00` | ❌ leave it alone |

You distinguish them in Ghidra (see below). With four use-sites of
`CRSF_BAUDRATE` in BF source (`crsfRxInit`, `crsfRxUpdateBaudrate`,
`getCrsfCachedBaudrate`, `getCrsfDesiredSpeed`), expect 1-4 *real*
literal-pool hits and maybe 0-2 collisions.

### Step 3: Verify in Ghidra (don't skip this)

Load `firmware.bin` per [[Loading Cortex-M Firmware in Ghidra]], jump
to each hit (`G` → `0x08000000 + offset`), and check what references
it:

- A real CRSF site looks like:
  ```
  ldr  r1, [PC, #0x14]      ; r1 = 0x66900
  ...
  bl   openSerialPort       ; ← passing it as the baud arg
  ```
- A collision has no `ldr` xref — it's just data inside some other
  blob.

If you don't have Ghidra handy, the cheap heuristic: a real literal
pool entry sits at a **4-byte aligned offset** inside `.text`,
typically within ~1 KB after a function that uses CRSF. Use
`arm-none-eabi-objdump -D -b binary -m arm --target=binary -Mforce-thumb firmware.bin | less` and search around each hit.

### Step 4: Compute the replacement bytes

```sh
python3 -c "print((921600).to_bytes(4, 'little').hex())"
# → 00100e00
```

So you're replacing `00 69 06 00` with `00 10 0E 00` at each verified
site.

### Step 5: Apply the patch

Pick *one* approach — they're equivalent:

```sh
# A) dd (precise, one site at a time):
printf '\x00\x10\x0e\x00' \
    | dd of=firmware.bin bs=1 seek=$((0x12345)) count=4 conv=notrunc

# B) Python (multi-site in one go, easier audit):
python3 - <<'PY'
import sys
sites = [0x12345, 0x23456, 0x34567, 0x45678]  # from Step 2, verified in Step 3
old = (420000).to_bytes(4, 'little')
new = (921600).to_bytes(4, 'little')
with open('firmware.bin', 'r+b') as f:
    for off in sites:
        f.seek(off)
        cur = f.read(4)
        if cur != old:
            sys.exit(f"FAIL at 0x{off:x}: got {cur.hex()}, expected {old.hex()}")
        f.seek(off)
        f.write(new)
        print(f"patched 0x{off:x}: {old.hex()} → {new.hex()}")
PY
```

### Step 6: Convert to `.hex` for flashing

```sh
arm-none-eabi-objcopy -I binary -O ihex \
    --change-addresses 0x08000000 \
    firmware.bin firmware_patched.hex
```

### Step 7: Flash and test

- BF Configurator: "Firmware Flasher" tab → "Load Firmware [Local]" →
  pick `firmware_patched.hex` → "Flash Firmware". Optionally enable
  "Full chip erase" first.
- Or STM32CubeProgrammer (SWD or DFU) → Program → select `.hex`.
- Or `dfu-util` with the `.bin` directly:
  `dfu-util -a 0 -s 0x08000000 -D firmware.bin`

Verify with a logic analyser on the RX UART: bit period should now be
≈1.085 µs (921 600 baud) instead of ≈2.381 µs (420 000 baud).

## When This *Doesn't* Work

### The constant was inlined as `movw/movt`

If the compiler emitted (for `mov r0, #420000`):

```
movw r0, #0x6900       ; encoded across bits of two halfwords
movt r0, #0x0006
```

…then the bytes `00 69 06 00` won't appear in flash as a contiguous
run — the immediate is **spread across the instruction encoding**
(imm4:i:imm3:imm8 fields). You can't grep for it.

The fix: disassemble (`arm-none-eabi-objdump -D -b binary
-m arm --target=binary -Mforce-thumb firmware.bin`), find the
`movw r0, #0x6900` instructions, and patch the encoded immediates.
Painful by hand; Ghidra's "Modify Instruction" + reassemble feature
handles it cleanly, as does radare2's `wa` (write assembly) command.

### Partial patching breaks the math

`CRSF_BAUDRATE` is used in **four code paths**, including the
frame-time computation:

```c
// rx/crsf.c:682
frameTimeNeededUs = (uint64_t)CRSF_TIME_NEEDED_PER_FRAME_US
                  * (uint64_t)CRSF_BAUDRATE  // ← if you patch this site only,
                  + (baudrate - 1)           //    timing math drifts
                  / baudrate;
```

If you flip only the `crsfRxInit` site, the UART runs at 921 600 but
the timeout / floor logic still thinks the link is 420 000. **Patch
all sites or none.**

### The image is signature-checked

Betaflight is **not** signed — byte-flipping works. But if you're
working on a different binary (e.g. a vendor's locked-down ESC firmware),
RSA/ECDSA verification at boot will reject the modified image. See
[[Cortex-M Binary Patching]] for the FPB live-patch alternative.

### The image carries a CRC

Some bootloaders verify a CRC32 over `.text` before jumping to
`main()`. Betaflight upstream does **not** do this for the
application image (it does CRC the config region, but that's
separate from code). Vendor-shipped images sometimes add one — if
your patched FC won't boot but the unpatched dump does, suspect a
CRC and either patch it out or recompute it.

## A Note on Endianness and Alignment

- STM32 is **little-endian** — every multi-byte value is stored
  low-byte-first in flash.
- Cortex-M `ldr [pc, #imm]` requires the literal to be **4-byte
  aligned**. Real literal-pool entries will always be at offsets
  divisible by 4. Hits at odd alignments are almost certainly
  collisions.
- The address arithmetic `pc + imm` rounds `pc` down to the nearest
  4-byte boundary (`pc` itself is `current_instruction + 4` in
  Thumb). When sanity-checking xrefs by hand, account for the round.

## Recipes Cheatsheet

```sh
# Convert dumped .bin → flashable .hex
arm-none-eabi-objcopy -I binary -O ihex --change-addresses 0x08000000 \
    firmware.bin firmware.hex

# Convert .hex → .bin (for dfu-util / hex-editing)
arm-none-eabi-objcopy -I ihex -O binary firmware.hex firmware.bin

# Compare two dumps byte-for-byte
cmp -l before.bin after.bin | head

# Disassemble raw .bin as Thumb-2 (rough triage; Ghidra is better)
arm-none-eabi-objdump -D -b binary -m arm --target=binary -Mforce-thumb \
    firmware.bin | less

# Verify the .hex carries the expected base address
head -1 firmware.hex
# expect :020000040800F2  (0x0800 in the high 16 bits)
```

## Gaps

> [!gap]
> Worked example needed: locate `CRSF_BAUDRATE` in a real BF F405 dump,
> screenshot the Ghidra xrefs, walk through patching all four sites
> end-to-end. Once done, file as a thesis-page case study and link
> back here.

> [!gap]
> Does the Betaflight bootloader (the MSP one Configurator flashes
> through, not the ST ROM DFU) impose any integrity check on the
> application image? If yes, patches need to update it.

## Sources

- Intel HEX spec: any of the canonical references; the four record
  types above cover 99 % of STM32 images.
- `arm-none-eabi-objcopy(1)` man page — `-I/-O` format names,
  `--change-addresses`.
- Companion pages: [[Cortex-M Firmware Dumping]],
  [[Cortex-M Binary Patching]],
  [[Loading Cortex-M Firmware in Ghidra]],
  [[Finding Defines in Firmware Binaries]],
  [[CRSF_BAUDRATE]],
  [[ARM Cortex-M Firmware Build Process]] (stage 4 = `objcopy`,
  which is the same tool you use here in reverse).
