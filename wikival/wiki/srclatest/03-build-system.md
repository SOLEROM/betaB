---
type: architecture
status: stable
created: 2026-05-14
updated: 2026-05-14
tags: [build, makefile, toolchain, platform, target, config]
source-commit: 6434dd725
---

# 03 — Build System

A single GNU Make invocation produces a flashable image for one of ~200 supported boards across 6 silicon families. The complexity is hidden behind a clean tri-axis model: **PLATFORM → TARGET → CONFIG**. This page walks that model and the Makefile machinery behind it.

## Three ways to invoke `make`

| Command | What happens |
|---------|--------------|
| `make TARGET=STM32F405` | Build a "generic F405" image with no board specialisation |
| `make CONFIG=MATEKF405TE` | Build a specific board (resolves to `TARGET=STM32F405` internally) |
| `make TARGET=SITL` | Build the SITL simulator as a native x86-64 binary |
| `make hex` / `make binary` / `make uf2` / `make exe` | Force a specific output format |
| `make EXST=yes …` | Build for external-storage-bootloader (H7 boards with bootloader flash) |
| `make make_unified_targets` / `make make_unified_targets_full` | Bulk-build all CI / preview configs |

Default goal: if `TARGET` or `CONFIG` is set, `.DEFAULT_GOAL := fwo` (the "firmware output" of the platform — `.hex` for STM32, `.exe` for SITL, `.uf2` for PICO). Otherwise `.DEFAULT_GOAL := all` (CI targets).

## The PLATFORM / TARGET / CONFIG triaxis

| Axis | Directory | Granularity | Set by |
|------|-----------|-------------|--------|
| **PLATFORM** | `src/platform/<arch>/` | Silicon vendor family (STM32 / AT32 / APM32 / ESP32 / PICO / SIMULATOR) | Auto-derived from TARGET |
| **TARGET**   | `src/platform/<arch>/target/<MCU>/` | Specific MCU (STM32F405, STM32H743, SITL) | `TARGET=…` or auto from CONFIG |
| **CONFIG**   | `src/config/configs/<BOARD>/` (separate submodule) | Specific flight controller board | `CONFIG=…` |

You can use TARGET without CONFIG to build a "generic" image for that MCU. You cannot use both at once (`Makefile` errors out).

```
$ make TARGET=STM32F405                  # generic F405 — no board hardware specifics
$ make CONFIG=MATEKF405TE                # MATEK F405 TE board (resolves TARGET=STM32F405)
$ make CONFIG=MATEKF405TE TARGET=...     # ERROR: TARGET or CONFIG should be specified. Not both.
```

## Top-level Makefile flow

`/mnt/betab/betaflight/Makefile` (~900 lines). Read it linearly and the flow is:

```
1. Capture user vars (TARGET, CONFIG, OPTIONS, EXST, DEBUG, RAM_BASED, ...)        lines  15–48
2. Compute paths (ROOT_DIR, SRC_DIR, PLATFORM_DIR, OBJECT_DIR, LIB_MAIN_DIR)         lines  60–72
3. Auto-hydrate submodules not marked update=none                                    lines  73–98
4. include mk/build_verbosity.mk                                                     line  104
5. include mk/system-id.mk     ← detect OS/arch                                      line  108
6. include mk/<OSFAMILY>.mk    ← linux.mk / macosx.mk / windows.mk                   line  143
7. include mk/tools.mk         ← define ARM_SDK_URL, GCC version, download rules     line  146
8. Resolve ARM_SDK_PREFIX / CROSS_CC / OBJCOPY / SIZE                                lines 152–166
9. include mk/preprocess.mk    ← provides pp_def_value() for header parsing           line  170
10. include mk/config.mk       ← if CONFIG=X, expand FC_TARGET_MCU → TARGET           line  179
11. include <TARGET_DIR>/target.mk                                                    line  195
12. include <TARGET_PLATFORM_DIR>/mk/<TARGET_MCU_FAMILY>.mk                           line  259
13. include mk/dronecan.mk     ← optional DroneCAN source fan-out                     line  262
14. include mk/source.mk       ← aggregate COMMON_SRC, MCU_COMMON_SRC, TARGET_SRC     line  263
15. Pattern rules for compile + link + objcopy                                        lines 542–600
```

Every variable you might want to override is set with `?=` so you can pass it on the command line: `make TARGET=STM32F405 DEBUG=GDB EXST=yes V=1`.

## What each `mk/*.mk` does

| File | Role |
|------|------|
| `build_verbosity.mk` | `V=0` (quiet, default) / `V=1` (echo every command) |
| `system-id.mk` | Detect host OS + arch, set `OSFAMILY=linux|macosx|windows` |
| `linux.mk` / `macosx.mk` / `windows.mk` | OS-specific helpers (path separators, archive tools) |
| `tools.mk` | Toolchain URLs + checksums. **Hard-pins `gcc-arm-none-eabi 13.3.Rel1`.** Downloads & extracts on demand into `tools/gcc-arm-none-eabi-13.3.rel1-.../`. |
| `tools_check.mk` | After MCU `.mk` loads, asserts the compiler exists and matches the required version |
| `preprocess.mk` | `pp_def_value(header.h, MACRO)` runs gcc preprocessor and evaluates a macro. Used to read `FC_TARGET_MCU`, `SYSTEM_HSE_MHZ`, `FC_VERSION_STRING` out of headers at make-time. |
| `config.mk` | If `CONFIG=X` is set, expand its `config.h` to derive `TARGET`, `HSE_VALUE`, `EXST_ADJUST_VMA`, and `CONFIG_REVISION`. Adds `src/config/configs/$(CONFIG)` to `INCLUDE_DIRS`. |
| `dronecan.mk` | If a platform opted in via `LIB_SUBMODULES += $(DRONECAN_LIB_DIR)`, add `lib/main/dronecan/libcanard/canard.c` and `io/dronecan/*.c` to `MCU_COMMON_SRC` |
| `source.mk` | The biggest one: builds the `SRC` variable. See §"Source aggregation". |
| `checks.mk` | Pre-build sanity checks (TARGET exists, CONFIG exists, mutually exclusive) |
| `openocd.mk` | `make openocd-gdb-<target>` helpers for flash/debug |

## Source aggregation (`mk/source.mk`)

The final `SRC` variable is the list of `.c` files passed to gcc. It's built in layers:

```
SRC := $(STARTUP_SRC)           ← MCU vendor startup_*.s (set by MCU .mk)
     + $(MCU_COMMON_SRC)        ← MCU family code (STM32F4 HAL/LL drivers)
     + $(TARGET_SRC)            ← from target.mk (per-MCU extras)
     + $(VARIANT_SRC)           ← variant-specific
     + $(FLASH_SRC)             ← drivers/flash/* if MCU supports it
     + $(MSC_SRC)               ← USB MSC if enabled
     + $(SDCARD_SRC)            ← drivers/sdcard/* if MCU supports it
     + $(COMMON_SRC)            ← src/main/* — the whole firmware
     - $(MCU_EXCLUDES)          ← things this MCU doesn't support
```

`COMMON_SRC` is ~280 files of platform-agnostic firmware (everything under `src/main/` except platform-specific stubs). `MCU_COMMON_SRC` is populated per-platform — for STM32F4, ~30 ST HAL driver `.c` files plus USB middleware.

**Per-file optimisation overrides** are explicit lists:

```makefile
SPEED_OPTIMISED_SRC += common/filter.c
SPEED_OPTIMISED_SRC += flight/imu.c
SPEED_OPTIMISED_SRC += flight/pid.c
SIZE_OPTIMISED_SRC  += cli/cli.c       ← huge file, optimise for size
SIZE_OPTIMISED_SRC  += osd/osd.c
NOT_OPTIMISED_SRC   += <a few debug files>
```

The compile pattern rule applies `-Ofast`, `-Os`, or default `-O2` (with LTO) accordingly. See `mk/source.mk:396-550`.

## Toolchain resolution

`mk/tools.mk` defines exactly one supported compiler version: **gcc-arm-none-eabi 13.3.Rel1**.

```
GCC_REQUIRED_VERSION = 13.3.1
ARM_SDK_URL (linux/x86_64)  = https://developer.arm.com/.../arm-gnu-toolchain-13.3.rel1-x86_64-arm-none-eabi.tar.xz
ARM_SDK_URL (macos x86_64)  = ...darwin-x86_64...
ARM_SDK_URL (macos arm64)   = ...darwin-arm64...
ARM_SDK_URL (windows)       = ...mingw-w64-i686...
```

MD5 checksums are checked after download. The toolchain lives in `tools/gcc-arm-none-eabi-13.3.rel1-…/bin/arm-none-eabi-*`. `make arm_sdk_install` triggers the download manually; otherwise it runs on first build.

If you've installed `gcc-arm-none-eabi` system-wide, the Makefile probes PATH first and uses that — but the version must match exactly, or `tools_check.mk` fails.

> **Repo-specific gotcha:** the `cross/` Dockerised build sidesteps the host toolchain entirely by providing the SDK at `/opt/gcc-arm-none-eabi-…` inside the container. The 4.5 build chain works around this by exporting `ARM_SDK_DIR=/tmp` to skip the download path. See `sitl/1boot.md` and `hw/formats.md` in the repo root.

`ccache` is auto-detected and prepended to `CROSS_CC` when present.

## Config resolution (`mk/config.mk`)

When `make CONFIG=BOARD` runs:

1. **Hydrate `src/config/`** if missing (it's a separate git repo).
2. Set `CONFIG_HEADER_FILE := src/config/configs/$(CONFIG)/config.h`.
3. Run `pp_def_value` on that header to extract `FC_TARGET_MCU` → assign to `TARGET`.
4. Same for `SYSTEM_HSE_MHZ` → `HSE_VALUE` (Hz, multiplied by 10⁶).
5. If `FC_VMA_ADDRESS` is defined, this is an EXST board — set `EXST=yes`, `EXST_ADJUST_VMA=<value>`.
6. Compile optional `src/config/configs/$(CONFIG)/config.c` if present.
7. Compute `CONFIG_REVISION` = git short hash of `src/config/` submodule.
8. Add `src/config/configs/$(CONFIG)` to `INCLUDE_DIRS` so the firmware sees the board's `config.h` ahead of any generic target headers.

The whole thing rides on `mk/preprocess.mk`:

```makefile
pp_def_value = $(strip $(shell echo "$(2)" | $(CROSS_CC) -E -imacros $(1) -P - 2>/dev/null))
```

i.e. "run the preprocessor with that header included, ask for the value of macro `$(2)`". Beautifully simple.

## Outputs

After `make TARGET=X` succeeds, you get (in `obj/`):

| Artifact | Built by | Used for |
|----------|----------|----------|
| `betaflight_<ver>_<target>.elf` | `CROSS_CC -o $@ ... $(LD_FLAGS) -T<linker.ld>` | GDB debugging, symbol extraction |
| `betaflight_<ver>_<target>.bin` | `objcopy -O binary $(ELF) $@` | Raw flash image (DFU `dfu-util -D`) |
| `betaflight_<ver>_<target>.hex` | `objcopy -O ihex --set-start 0x08000000 $(ELF) $@` | Intel HEX (Configurator flash, ST-Link) |
| `betaflight_<ver>_<target>.dfu` | `src/utils/dfuse-pack.py $(HEX)` | DfuSe format (STM32 USB DFU) |
| `betaflight_<ver>_<target>.uf2` | `picotool uf2 convert` | Drag-and-drop flash (Pico only) |
| `betaflight_<ver>_<target>.map` | `-Wl,-Map=$@` | Memory map for size analysis |
| `betaflight_<ver>_<target>.lst` | `objdump -S` | Mixed source/asm listing |
| `betaflight_<ver>_<target>` (no ext) | native gcc | SITL native executable |

**STM32 flash base is `0x08000000`** — this is why `--set-start 0x08000000` appears in the hex rule. The `bin` format has no address info, so converting bin→hex without `--change-addresses 0x08000000` is a classic foot-gun (see `hw/formats.md` in this repo).

### EXST builds

For boards with a separate bootloader flash partition, `EXST=yes` enables a post-link pipeline:

1. Link the firmware as usual.
2. Pad the `.bin` to `FIRMWARE_SIZE` with zeros.
3. Compute MD5 over the padded image.
4. Embed the MD5 at offset `(1024 * FIRMWARE_SIZE) - 16`.
5. Patch the MD5 back into the ELF's `.exst_hash` section.
6. Re-export the `.hex` from the patched binary with `--adjust-vma`.

This lets the bootloader verify integrity at boot, and lets the firmware live at a non-zero offset within flash.

## SITL build differences

`make TARGET=SITL` picks `src/platform/SIMULATOR/mk/SITL.mk`. Key differences:

- `PLATFORM_SDK := none` — no ARM toolchain, native gcc/clang.
- `ARCH_FLAGS` empty — host CPU.
- Drops `--specs=nano.specs`, adds `-lm -lpthread -lc -lrt` (Linux).
- Virtual drivers replace real hardware: `accgyro_virtual.c`, `barometer_virtual.c`, `compass_virtual.c`, `serial_tcp.c`, `gps_virtual.c`, `blackbox_virtual.c`.
- `LD_SCRIPT` is a fake `sitl.ld` used only for symbol ordering — actual linking is native.
- macOS ARM64 has a separate workaround block (`SITL.mk:67–82`) that drops various flags Apple Silicon's linker chokes on.
- Default output is `exe` (not `hex`). Run with `./obj/betaflight_<ver>_SITL`.
- TCP port 5760 for MSP (UART1 → SITL TCP), 5761 for the second serial, etc.

## Per-platform Makefile fragments

Each platform's `mk/<MCU_FAMILY>.mk` (e.g. `src/platform/STM32/mk/STM32F4.mk`) defines:

- `MCU_COMMON_SRC` — the ST HAL/LL driver `.c` files specific to that family.
- `STARTUP_SRC` — vendor `startup_stm32f405xx.s` etc.
- Compiler flags: `-mcpu=cortex-m4 -mthumb -mfloat-abi=hard -mfpu=fpv4-sp-d16` (F4)
                  / `-mcpu=cortex-m7 -mfloat-abi=hard -mfpu=fpv5-d16` (H7), etc.
- `DEVICE_FLAGS` such as `-DSTM32F405xx -DUSE_HAL_DRIVER`.
- `LD_FLAGS` MCU-specific bits.
- `INCLUDE_DIRS` for vendor SDK headers.

`src/platform/STM32/mk/STM32_COMMON.mk` holds the shared bits used by every STM32 subfamily.

## Submodules

`.gitmodules` distinguishes auto-hydrated submodules from opt-in ones via `update = none`:

**Auto (always present):**
- `src/config` → board configs
- `lib/main/dronecan/libcanard` → DroneCAN transport

**Opt-in (only fetched when their platform builds):**
- `lib/main/pico-sdk`
- `lib/main/esp-idf`
- `lib/main/STM32H5`, `STM32C5`, `STM32N6` (newer STM32 lines)
- `lib/main/APM32F4`
- `lib/main/GD32H7`, `lib/main/X32M7`

The auto-hydration is implemented inline in the Makefile (lines 73–98) using `awk` over `.gitmodules`. Targets that need opt-in submodules invoke their own hydration rules in `make help` (e.g. `make pico_sdk_install`).

## Useful Make targets (`make help`)

| Target | Effect |
|--------|--------|
| `make help` | Full help — list of overrideable vars and targets |
| `make targets` | List every TARGET available |
| `make configs` | List every CONFIG available (hydrates `src/config/`) |
| `make clean_<TARGET>` | Clean one target's `obj/main/<TARGET>/` |
| `make clean` | Clean everything in `obj/` |
| `make arm_sdk_install` | Force-download the ARM toolchain |
| `make openocd-gdb-<TARGET>` | Flash + GDB session over OpenOCD |
| `make all` | Build all CI targets (slow) |
| `make all_<PLATFORM>` | All targets for one platform |
| `make make_unified_targets` | All CONFIGs (slow) |
| `make CFLAGS_DISABLED='-Werror'` | Strip a flag globally for a one-off build |

## End-to-end example: `make CONFIG=MATEKF405TE hex`

```
1. Makefile parses CONFIG=MATEKF405TE
2. checks.mk passes (no TARGET conflict)
3. config.mk runs gcc preprocessor on src/config/configs/MATEKF405TE/config.h
   → FC_TARGET_MCU = STM32F405 → TARGET assignment
   → SYSTEM_HSE_MHZ = 8 → HSE_VALUE = 8000000
4. Makefile resolves TARGET_PLATFORM=STM32 via wildcard match
   on src/platform/STM32/target/STM32F405/target.mk
5. include src/platform/STM32/target/STM32F405/target.mk
   → adds TARGET_SRC, LD_SCRIPT = src/platform/STM32/link/stm32_flash_f405.ld
6. include src/platform/STM32/mk/STM32F4.mk
   → MCU_COMMON_SRC = ST HAL drivers
   → DEVICE_FLAGS = -DSTM32F405xx ...
7. source.mk runs → SRC list assembled
8. For each *.c in SRC:
   $(CROSS_CC) -c -O2 -flto -mcpu=cortex-m4 ... -o obj/main/STM32F405_MATEKF405TE/foo.o foo.c
9. $(CROSS_CC) -o betaflight_2026.6.0_STM32F405.elf $(all .o) $(LD_FLAGS)
10. objcopy -O ihex --set-start 0x08000000 → obj/betaflight_2026.6.0_STM32F405_MATEKF405TE.hex
```

That hex is what Configurator's Firmware Flasher writes to the FC.

## See also

- [[02-directory-layout]] for the directory tree the build operates on
- [[07-hal-and-drivers]] for what the platform-specific drivers actually do
- [[12-config-and-pg]] for how board `config.h` USE_* macros gate features at compile time
- `hw/formats.md` in the repo root for elf/hex/bin conversion details and pitfalls
