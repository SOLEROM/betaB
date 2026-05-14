---
type: concept
title: "Finding Defines in Firmware Binaries"
status: developing
domain: build-system
created: 2026-05-12
updated: 2026-05-12
tags: [concept, build-system, toolchain, reverse-engineering, debugging]
---

# Finding Defines in Firmware Binaries

## Definition
The practical question: *given a `#define` in the source, where does its value end up in the `.elf` / `.hex` / `.bin`?* The trap is the framing — a `#define` is **textual substitution at preprocess time** ([[ARM Cortex-M Firmware Build Process]] stage 1), not a thing the linker places. By stage 3 the symbol `MY_DEFINE` no longer exists; only its expanded use-sites do. So "where it lives in the binary" depends entirely on *how it was used in source*.

## The Decision Tree

| Use of the define | Where it lands | How to find it |
|---|---|---|
| `#ifdef USE_GPS` / `#if FOO` | Presence/absence of whole code blocks | Diff two builds (`USE_GPS=ON` vs `OFF`); or `arm-none-eabi-gcc -E` to confirm which branch survived |
| `#define MAX_MOTORS 4` used as array size | Allocation in `.bss` / `.data` | Symbol's size in `.map` file |
| `#define PID_HZ 8000` used in math/comparisons | Inlined **immediate operand** in code (`mov`, `movw/movt`, `cmp`) | `arm-none-eabi-objdump -d` and search use-sites |
| `#define BANNER "BF"` (string literal) | Bytes in `.rodata` | `strings firmware.elf` |
| `#define FOO 0x12345678` used to init a global | Bytes in `.data` LMA region (flash) | `.map` gives the symbol address; `objdump -s -j .data` shows bytes |
| `#define REG_BASE 0x40021000` (peripheral addr) | Immediate in `ldr`/`str` instructions | `objdump -d` — appears as literal pool constant |

Key insight: **a numeric define used in three different files appears in three different places** in the binary, with no symbol tying them together. The C preprocessor erases the name before the compiler ever sees it.

## Tooling, Ordered by Usefulness

### 1. Preprocessor expansion (most direct)
See exactly what the compiler saw after macro substitution:

```sh
# Inside the cross env (cd /mnt/betab/cross && make shell)
arm-none-eabi-gcc -E -dM src/main/foo.c | grep MY_DEFINE     # all defines visible
arm-none-eabi-gcc -E src/main/foo.c | less                    # full preprocessed source
```

Or build with `EXTRA_FLAGS="-save-temps"` so every `.o` drops a `.i` (preprocessed) and `.s` (assembly) next to it. Best one-shot way to confirm "did this `#ifdef` branch make it in?"

### 2. DWARF macro info (when built with `-g3`)
The DWARF section can retain macro names + source locations:

```sh
arm-none-eabi-readelf --debug-dump=macro obj/main/betaflight_STM32F405.elf | grep MY_DEFINE
```

Requires `-g3` (not just `-g`). Betaflight's default build doesn't enable this — add to CFLAGS if you want it.

### 3. Disassembly with source interleave
Catches numeric defines baked in as immediates:

```sh
arm-none-eabi-objdump -d -S obj/main/betaflight_STM32F405.elf | less
arm-none-eabi-objdump -d obj/main/betaflight_STM32F405.elf | grep -B2 '#8000'   # specific value
```

`-S` interleaves source lines if `-g` was used. Look for `movw r0, #LOW`, `movt r0, #HIGH` pairs (large constants) or `mov r0, #imm` (small).

### 4. The `.map` file
Only useful for defines that affect *symbol allocation* — array sizes, struct sizes, named globals/constants:

```sh
grep MY_DEFINE obj/main/betaflight_STM32F405.map
grep ' \.text\.' obj/main/betaflight_STM32F405.map | sort -k3 -r | head -20   # biggest .text symbols
```

The `.map` will not list `PID_HZ` if the define was only used as a literal `8000` inside expressions.

### 5. `strings` for text defines
String-valued defines (banners, format strings, version stamps):

```sh
arm-none-eabi-strings obj/main/betaflight_STM32F405.elf | grep -i mydefine
arm-none-eabi-strings -tx obj/main/betaflight_STM32F405.elf | grep ' BF '   # with offsets
```

### 6. `objcopy -O verilog` / `nm` for symbol tables
For named globals initialised from a define:

```sh
arm-none-eabi-nm --print-size --size-sort obj/main/betaflight_STM32F405.elf | tail
arm-none-eabi-nm -S obj/main/betaflight_STM32F405.elf | grep my_global
```

### 7. Diff-build (the brute-force confirmation)
For `USE_X` feature gates: build twice, diff the `.map` or `.bin`:

```sh
make build TARGET=STM32F405                              # baseline
arm-none-eabi-size obj/main/betaflight_STM32F405.elf     # text/data/bss numbers
# rebuild with the define flipped...
diff baseline.map variant.map | less
```

Section sizes shift by exactly the bytes the feature added. Often the fastest way to *prove* a define did something.

## In Betaflight

- **Configurator settings ≠ `#define`s.** Almost everything you tweak in the BF Configurator (PIDs, rates, modes, motor map) lives in the **PG (Parameter Group) system**: structs initialised by `pgResetFn_*` functions, stored in flash config region, mutated at runtime via CLI/MSP. To find these, search `.map` for `pgResetFn_` or `pgRegisterFn_` — the names survive linking.
- **Real `#define`s in BF** tend to be: `USE_*` feature gates (Cloud Build flips these), `MAX_SUPPORTED_*` array sizes, target-pin assignments (`MOTOR1_PIN PA0`), and clock/peripheral base addresses from CMSIS headers.
- **Target pin defines** — e.g. `#define MOTOR1_PIN PA0` — disappear into `ioTag_t` table initialisers in `.rodata`. Cross-ref with the `target.c` file and search the `.map` for the table symbol.
- The [[Cloud Build System]] toggles `USE_*` defines server-side before invoking the same toolchain.

## Worked Example (typical workflow)

You want to know "is `USE_GPS` actually in my F405 build, and what did it cost in flash?":

```sh
# 1. Confirm it's defined
arm-none-eabi-gcc -E -dM -DSTM32F405 src/main/main.c | grep USE_GPS

# 2. Find GPS symbols that survived linking
grep -i gps obj/main/betaflight_STM32F405.map | head -30

# 3. Sum the flash cost
grep -i gps obj/main/betaflight_STM32F405.map | awk '{s+=strtonum($3)} END {print s}'

# 4. Confirm by diff-build with -DUSE_GPS=0
```

## Key Relationships
- Stage in pipeline: [[ARM Cortex-M Firmware Build Process]] (defines are erased at stage 1)
- Inverse direction (no source available): [[Loading Cortex-M Firmware in Ghidra]] — finding *values* in the binary without knowing the names
- Patching values post-build: [[Cortex-M Binary Patching]]
- Build options the Cloud system toggles: [[Cloud Build System]]
- Inspecting BF runtime config (not defines): MSP Get/Set commands in [[MSP Protocol]]

> [!key-insight]
> The C preprocessor is destructive: by the time the linker runs, every `#define` is anonymous data scattered across `.text`, `.data`, `.rodata`, or simply gone (if `#ifdef`'d out). You can't grep the `.hex` for `MY_DEFINE` — you have to grep for what `MY_DEFINE` *expanded into*, and reason backwards from how it was used.

## Sources
- [[memfault-linker-scripts]] — explains how the `.map` is built and what survives linking
- [[aticleworld-stm32-build]] — the five-stage pipeline (preprocess → compile → link → objcopy → flash)
- [[wrongbaud-ghidra-stm32-loader]] — the dual problem: finding constants in a binary you didn't compile
