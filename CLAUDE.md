---
noteId: "da3ded204f9c11f194a2c3b1eecd91b7"
tags: []

---

# betaB — Betaflight Playground

A learning/research repo for exploring [Betaflight](https://betaflight.com/), the open-source flight control firmware for STM32-based multirotors.

## Layout

```
betab/
├── betaflight/      # upstream master submodule — read-only reference
├── betaflight_4.5/  # 4.5-maintenance checkout — GUI work with Configurator 10.10.0
├── sitl/            # Dockerised SITL builder/runner (native x86-64 BF emulator)
├── cross/           # Dockerised ARM cross-compile (produces .elf/.hex/.bin for STM32 targets)
├── hw/              # docs: reading firmware off real FCs + the ELF/hex/bin trinity
├── configurator/    # vendored Configurator 10.10.0 .deb + deps
├── wikival/         # Obsidian research vault (notes, concepts, reverse-engineering)
└── README.md
```

- **`betaflight/`** / **`betaflight_4.5/`** — upstream source-of-truth. Don't modify. Use 4.5 for GUI work (10.10.0 Configurator complains about master's `26.6.0` version string). Master is fine for CLI-only research.
- **`sitl/`** — `make image / build / run`. SITL binary at `<src>/obj/main/betaflight_SITL.elf`. Configurator connects at `tcp://127.0.0.1:5761` (UART1). See `sitl/README.md` and `sitl/1boot.md` for the port off-by-one, the `ARM_SDK_DIR=/tmp` workaround on 4.5, and the `set motor_pwm_protocol = PWM` first-boot step.
- **`cross/`** — ARM toolchain in Docker. Builds real-firmware artifacts under `betaflight/obj/` (e.g. `betaflight_2026.6.0-alpha_STM32F405.{elf,hex,bin}`).
- **`hw/`** — `formats.md` (ELF/hex/bin views + `objcopy` conversions, incl. `--change-addresses 0x08000000` for bin→hex) and `dumps.md` (DFU/SWD/UART firmware readback + RDP gotcha).
- **`wikival/`** — knowledge base. See `wikival/CLAUDE.md` for vault conventions, note types, and `/wiki`-family workflows (ingest, query, lint, save, research).

## Current scope

Working end-to-end loop is in place: edit a `#define` in BF source → cross-compile or SITL-compile → observe the change in Configurator (or by binary-patching the resulting `.bin`). Validated round-trip via `FC_VERSION_PATCH_LEVEL` in `version.h` showing up in the Configurator header.

Active threads:
- Reverse-engineering exercises: locate constants in `.rodata` vs immediates in `.text`, patch them in `.bin`, convert back to `.hex`, flash and verify.
- Reading hardware firmware via `dfu-util` and diffing against our cross-built artifacts.
- Building out the wiki as we go.

## Working tips for next session

- Wiki work → read `wikival/CLAUDE.md` first.
- Code reading → `betaflight_4.5/src/main/` for 4.5 features; `betaflight/src/main/` for master.
- Don't trust old eeprom claims: 4.5's `target.h` sets `EEPROM_SIZE 32768` (not 8192 as `sitl/1boot.md` still says — TODO fix).
- Submodule init: `git submodule update --init --recursive` if `betaflight/` is empty.
- STM32 flash base is `0x08000000` (F4/F7/G4/H7 bank1). H7 bank2 is `0x08100000`. Bin→hex without `--change-addresses` is a foot-gun.
