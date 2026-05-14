---
type: how-to
status: stable
created: 2026-05-14
updated: 2026-05-14
tags: [modification, cookbook, how-to, contributing]
source-commit: 6434dd725
---

# 13 — Modification Guide

A reverse pointer from "I want to change X" to "edit file Y". Use this once you've read enough of the previous pages to know the layout.

Conventions: file paths are absolute from `/mnt/betab/betaflight/`. Edits in `src/main/` are platform-agnostic; edits in `src/platform/<arch>/` are MCU-specific.

## Setup checklist before any change

1. **Pick the right tree.** Master is in `betaflight/` (this page). 4.5 maintenance is in `betaflight_4.5/`. GUI work that talks to Configurator 10.10.0 needs 4.5 (master's `26.6.0` version string upsets Configurator).
2. **Initialise submodules.** `git submodule update --init --recursive`. Otherwise `src/config/` is empty and `make CONFIG=X` fails.
3. **Have the toolchain.** Either let `make` download it (`tools/gcc-arm-none-eabi-13.3.rel1-…`) or use the `cross/` Docker setup in this repo.
4. **Build the baseline first.** `make CONFIG=<your_board>` — make sure it builds clean before changing anything. Note the firmware size from the linker output for comparison.

## "I want to change…"

### …a flight characteristic (PID gains / rates / filter cutoffs)

You almost never touch source for this — change settings via CLI or Configurator. They're persisted in PGs (see [[12-config-and-pg]]).

If you want to change the **default** for a board's first-boot experience, edit `src/config/configs/<BOARD>/defaults.txt` (a list of `set name=value` lines applied on first boot).

If you want to change the **global default** in code, edit the `PG_RESET_TEMPLATE` in the relevant `src/main/pg/<feature>.h`. Example: `src/main/pg/pid.h` for default PID gains. Bumping the PG version forces existing configs to reset.

### …how a setting maps to behaviour

The setting lives in a PG (read with `<feature>Config()->fieldName`). Find where it's consumed by:

```
grep -r "settingName" src/main/
```

Example: `gyro_lpf1_static_hz` is consumed in `src/main/sensors/gyro_init.c` to initialise the LPF1 filter.

### …add a new CLI setting

Worked example: add a setting `crash_recovery_p` that's editable.

1. Add the field to the relevant PG (here `src/main/pg/pid.h`):

   ```c
   typedef struct pidProfile_s {
       // ... existing fields ...
       uint8_t crash_recovery_p;
   } pidProfile_t;
   ```

2. Set a default in the PG's reset:

   ```c
   PG_RESET_TEMPLATE(pidProfile_t, pidProfile,
       // ...
       .crash_recovery_p = 30,
   );
   ```

3. (Optional but recommended) Bump the PG version — find the `PG_REGISTER_*` call for `PG_PID_PROFILE` and increment its version argument. Skip this if you're appending to the end and want forward-compat.

4. Add a settings table entry in `src/main/cli/settings.c`:

   ```c
   { "crash_recovery_p", VAR_UINT8 | PROFILE_VALUE, .config.minmax = { 0, 100 },
     PG_PID_PROFILE, offsetof(pidProfile_t, crash_recovery_p) },
   ```

5. Read it from flight code:

   ```c
   const pidProfile_t *p = currentPidProfile;
   uint8_t val = p->crash_recovery_p;
   ```

Build, flash, `set crash_recovery_p=50`, `save`. Done.

### …expose that setting to MSP (so Configurator can edit it)

Find the appropriate `MSP_<FEATURE>` / `MSP_SET_<FEATURE>` opcode in `src/main/msp/msp.c`. For a PID-profile field, that's `MSP_PID_ADVANCED` and `MSP_SET_PID_ADVANCED`. Add reads and writes:

```c
// MSP_PID_ADVANCED reply
sbufWriteU8(dst, currentPidProfile->crash_recovery_p);

// MSP_SET_PID_ADVANCED
currentPidProfile->crash_recovery_p = sbufReadU8(src);
```

Update Configurator side similarly (out of scope for this repo).

### …expose to CMS (RC-stick OSD menu)

Pick the right `cms_menu_*.c` (here `cms_menu_imu.c`) and add an `OSD_Entry`:

```c
{ "CRASH RECOVERY P", OME_UINT8, NULL,
  &(OSD_UINT8_t){ &currentPidProfile->crash_recovery_p, 0, 100, 1 }, 0 },
```

The CMS entry binds directly to the struct field — no settings-table indirection.

### …add a new flight mode (BOX)

1. Add a new value to `boxId_e` in `src/main/fc/rc_modes.h`. Append at the end to keep IDs stable.
2. Add a name string in the box table (`boxes[]` in `src/main/msp/msp_box.c`). The name appears in Configurator's modes tab.
3. If the mode should set a bit in `flightModeFlags`, add the bit to `flightModeFlags_e` in `src/main/fc/runtime_config.h` and wire it in `src/main/fc/rc_modes.c::updateActivatedModes()`.
4. Implement the mode's behaviour. Reading `IS_RC_MODE_ACTIVE(BOXYOURMODE)` or `FLIGHT_MODE(YOUR_MODE)` from wherever the behaviour belongs.

### …add a new MSP command

1. Pick a new opcode number — use the next free integer in `src/main/msp/msp_protocol_v2_betaflight.h` (for Betaflight-specific commands). Avoid colliding with INAV or MultiWii common opcodes.
2. Add the handler in `src/main/msp/msp.c::mspFcProcessCommand()`'s big switch. For sets, also add to `mspFcProcessOutCommand()` or the appropriate split function.
3. Read input via `sbufRead*(src)`, write output via `sbufWrite*(dst)`.
4. Return `MSP_RESULT_ACK` / `MSP_RESULT_ERROR` / `MSP_RESULT_NO_REPLY`.

Configurator side will need a parallel implementation.

### …add a new target / board

For an existing MCU family:

1. Pick a board name (uppercase short, e.g. `MYBOARD_F405`).
2. In the `src/config/` submodule (it's a separate repo), create `configs/MYBOARD_F405/`. Inside:
   - `config.h` — set `FC_TARGET_MCU`, `BOARD_NAME`, `MANUFACTURER_ID`, `SYSTEM_HSE_MHZ`, and pin assignments via `#define` macros.
   - `defaults.txt` (optional) — first-boot CLI defaults.
   - `config.c` (optional) — custom init code.
3. Commit + push to your config fork.
4. Build with `make CONFIG=MYBOARD_F405`.

For a brand-new MCU family (e.g. a new STM32 series ST just released):

1. Create `src/platform/STM32/target/<NEW_MCU>/target.{h,mk}`.
2. Create a linker script under `src/platform/STM32/link/stm32_flash_<new>.ld` based on a similar existing MCU's script.
3. Create or extend `src/platform/STM32/mk/<NEW_MCU_FAMILY>.mk` with vendor SDK paths, compiler flags (`-mcpu=…`, FPU type, vendor `#define`s).
4. Add the vendor HAL/LL SDK as a submodule under `lib/main/STM32<NEW>/`. Reference it from the MK file. Consider `update = none` if it's heavy.
5. Add the new MCU's startup `.s` file under `src/platform/STM32/startup/`.
6. Add concrete peripheral drivers (`adc_<new>.c`, `bus_spi_<new>.c`, `timer_<new>.c`, etc.) — usually copy-and-adapt from the closest existing subfamily.
7. Build with `make TARGET=<NEW_MCU>` to shake out missing pieces.

For a brand-new vendor entirely (e.g. a new RISC-V SoC): same pattern but create `src/platform/<NEW_ARCH>/` from scratch, with its own `mk/`, `link/`, `startup/`, `target/`, `include/`, and drivers.

### …add a new sensor (gyro/acc/baro/mag)

Worked example: a new SPI gyro called `ACME_G1000`.

1. Add `src/main/drivers/accgyro/accgyro_spi_acme_g1000.c/.h`. Implement:
   - `bool acmeG1000SpiDetect(const extDevice_t *dev)` — read WHO_AM_I.
   - `void acmeG1000SpiAccInit(accDev_t *acc)` and `void acmeG1000SpiGyroInit(gyroDev_t *gyro)`.
   - Read function pointers stored on `gyroDev_t::readFn`.
2. Add an enum to `gyroHardware_e` in `src/main/drivers/accgyro/accgyro.h`.
3. Add the probe call in `src/main/sensors/gyro_init.c::gyroDetect()` and `accDetect()`.
4. Add a `USE_GYRO_SPI_ACME_G1000` guard around includes/probes.
5. Define `USE_GYRO_SPI_ACME_G1000` in the `config.h` of any board that has the chip.

For barometers / compasses the recipe is identical with the corresponding `*Detect()` and `*Init()` functions in `sensors/barometer.c` / `sensors/compass.c`.

### …support a new RX protocol

If serial (UART byte stream):

1. Create `src/main/rx/<myproto>.c/.h`.
2. Implement `bool <myproto>Init(const rxConfig_t *config, rxRuntimeState_t *state)`. Populate `rcFrameStatusFn`, `rcReadRawFn`, `rcGetFrameTimeUsFn` on `state`.
3. Open a serial port via `openSerialPort(FUNCTION_RX_SERIAL, ...)` with appropriate baud/parity/inversion.
4. Add an enum value to `rxConfig_t::serialrx_provider`.
5. Add the dispatch case in `src/main/rx/rx.c::serialRxInit()`.

If SPI-based:

1. Create `src/main/rx/<myproto>.c/.h` and use `rxSpi*` helpers.
2. Add an enum value to `rx_spi_protocol_e` in `rx_spi.h`.
3. Wire `rxSpiDeviceInit()` in `rx_spi.c` to call your init for that enum.
4. If the chip is new, also add a `drivers/rx/<chip>.c` family driver.

### …change scheduler behaviour / add a new task

1. Add a task entry to `taskId_e` in `src/main/fc/tasks.h`.
2. Add a `DEFINE_TASK(…)` entry in `tasks[]` in `src/main/fc/tasks.c`.
3. Implement the task function.
4. In `fc/init.c`, call `setTaskEnabled(TASK_MYNEW, true)` if the task should run.
5. Pick a priority carefully — never `TASK_PRIORITY_REALTIME` unless you also accept responsibility for not blowing the gyro deadline.

If the task needs to be conditional (e.g. only when a feature is on), gate the `setTaskEnabled()` call on `featureIsEnabled(FEATURE_X)`.

### …extend OSD with a new element

1. Add an enum value to `osd_items_e` in `src/main/osd/osd_elements.h`.
2. Implement the render function `osdElementRender_<name>(osdElementParms_t *e)` in `src/main/osd/osd_elements.c`.
3. Register it in the `osdElementDrawFunction[]` table in `osd_elements.c`.
4. Add a default position in `src/main/pg/osd.h` reset template (or set position 0 = invisible by default).
5. Optionally add a name string for Configurator's OSD layout tool.

### …extend telemetry (new field in an existing protocol)

Find the protocol file in `src/main/telemetry/` (e.g. `crsf.c`). Each protocol's `process` function builds frames at scheduled intervals. Add your data to the appropriate "sensor frame" assembler. The receiving transmitter side must also know about the new field; without OpenTX/EdgeTX support, custom fields show as `<unknown>`.

### …reduce firmware size (build won't fit in flash)

Compile-time options to strip in board `config.h`:

```c
#undef USE_BLACKBOX
#undef USE_GPS_RESCUE
#undef USE_OSD_HD            // keep MAX7456 only
#undef USE_DYN_NOTCH_FILTER
#undef USE_RX_EXPRESSLRS
#undef USE_VTX_TRAMP         // keep just one VTX protocol
#undef USE_TELEMETRY_HOTT    // strip rare telemetry
#undef USE_NRF24_RX          // strip toy-grade RX
```

Each one drops kilobytes. The `cli/cli.c` file is the single biggest contributor at ~85 KB; `USE_CLI_BATCH` and similar gates trim it but rarely enough to matter. Look at the `.map` file for the actual size accounting.

### …debug a crash / hardfault

1. Enable hardfault diagnostics: `make DEBUG_HARDFAULTS=1` (don't fly a build with this — it disables PWM output on fault).
2. On crash, the FC writes register state to backup-domain registers and resets. Boot log shows `HardFault at PC=...`.
3. Cross-reference PC with the `.lst` file (`objdump -S` output) to find the offending line.
4. For deeper analysis: `make DEBUG=GDB`, attach OpenOCD + GDB.

### …investigate a bug with blackbox

1. `set blackbox_device = SERIAL` (or flash/SDCARD).
2. `set blackbox_sample_rate = 1` for max-rate logging.
3. Reproduce, retrieve log, open in Blackbox Explorer.
4. Plot the trace that's misbehaving. Note timestamps. Cross-reference to source: which subtask runs at that rate, what input did it have, what should the output have been.

### …add a new MCU peripheral (e.g. a new SPI controller)

1. Add an enum entry to `SPIDevice` (typically in `bus_spi.h`).
2. Add concrete init code in `src/platform/<arch>/bus_spi_<flavour>.c`.
3. Add a target `target.h` `USE_SPI_DEVICE_<N>` macro.
4. Wire it into the resource registry / pin-assignment via PG `spiPinConfig_t` defaults.

### …port to a different RTOS / OS

Don't. Betaflight isn't structured to run under a preemptive RTOS — the scheduler model is fundamental. If you need OS abstraction, look at PX4 or ArduPilot instead.

The one supported "OS-ish" target is **SITL** (Linux/macOS native) — see `src/platform/SIMULATOR/`.

## When in doubt — file lookup table

| I want to change… | Edit |
|--------------------|------|
| PID math | `src/main/flight/pid.c` |
| Mixer math | `src/main/flight/mixer.c` |
| Attitude estimation | `src/main/flight/imu.c` |
| Gyro filtering | `src/main/sensors/gyro.c` + `gyro_filter_impl.c` |
| RC stick interpretation | `src/main/fc/rc.c` |
| RC mode activation | `src/main/fc/rc_modes.c` |
| Arming gate | `src/main/fc/core.c::tryArm()`, `src/main/fc/runtime_config.c` |
| Task table | `src/main/fc/tasks.c` |
| Scheduler | `src/main/scheduler/scheduler.c` |
| Boot init order | `src/main/fc/init.c` |
| MSP opcodes | `src/main/msp/msp.c`, `msp_protocol*.h` |
| CLI commands | `src/main/cli/cli.c` |
| CLI settings | `src/main/cli/settings.c` |
| CMS menus | `src/main/cms/cms_menu_*.c` |
| OSD elements | `src/main/osd/osd_elements.c` |
| Blackbox fields | `src/main/blackbox/blackbox_fielddefs.h` |
| Telemetry protocol | `src/main/telemetry/<protocol>.c` |
| RX protocol | `src/main/rx/<protocol>.c` |
| Sensor probe | `src/main/sensors/<sensor>.c` + `sensors/<sensor>_init.c` |
| Sensor driver | `src/main/drivers/accgyro/`, `barometer/`, `compass/` |
| Motor protocol | `src/main/drivers/motor.c`, `dshot*.c`, `pwm_output*.c` |
| Bus driver (abstract) | `src/main/drivers/bus_spi.h`, `bus_i2c.h` |
| Bus driver (platform) | `src/platform/<arch>/bus_spi_*.c`, `bus_i2c_*.c` |
| Linker layout | `src/platform/<arch>/link/*.ld` |
| Toolchain version | `mk/tools.mk` |
| Build flags | `mk/source.mk`, MCU's `mk/<MCU>.mk` |
| Board pins | `src/config/configs/<BOARD>/config.h` |
| Default settings per board | `src/config/configs/<BOARD>/defaults.txt` |
| Failsafe behaviour | `src/main/flight/failsafe.c` |
| GPS rescue | `src/main/flight/gps_rescue_*.c` |
| Anti-gravity / iTerm relax / TPA | `src/main/flight/pid.c` (within `pidController`) |
| Dynamic notch filter | `src/main/flight/dyn_notch_filter.c` |
| RPM filter | `src/main/flight/rpm_filter.c` |
| Beeper patterns | `src/main/io/beeper.c` |
| LED strip modes | `src/main/io/ledstrip.c` |
| Parameter Groups | `src/main/pg/*.h` |

## Conventions for upstream-friendly changes

If you want to contribute back:

- **No tabs.** 4-space indents. The codebase uses `clang-format` rules; running it before commits avoids whitespace churn.
- **Keep diffs minimal.** Betaflight reviewers are stretched. Massive refactors with new features are unlikely to merge.
- **Guard new features behind a `USE_*` flag.** If your feature increases firmware size, gate it so other boards opt out.
- **Test on real hardware.** SITL is not enough for a flight-control change. Mention the test board in the PR description.
- **Don't bump PG version unnecessarily.** Forward-compatible adds are preferred — append fields with templated defaults.

## See also

- [[03-build-system]] for build mechanics
- [[12-config-and-pg]] for PG version bumping rules
- [[07-hal-and-drivers]] for porting layer geometry
- `CONTRIBUTING.md` at the repo root for the upstream contribution policy
