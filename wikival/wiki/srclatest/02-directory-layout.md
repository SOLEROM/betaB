---
type: architecture
status: stable
created: 2026-05-14
updated: 2026-05-14
tags: [layout, structure, tree]
source-commit: 6434dd725
---

# 02 — Directory Layout

Repo root: **`/mnt/betab/betaflight/`** (commit `6434dd725`).

```
betaflight/
├── Makefile               ← single entrypoint, drives all builds
├── mk/                    ← Makefile fragments (~12 files)
├── src/
│   ├── main/              ← the firmware itself, platform-agnostic
│   ├── platform/          ← MCU-family-specific code & SDKs
│   ├── config/            ← submodule: per-board config headers
│   ├── test/              ← host-side GoogleTest harness
│   └── utils/             ← Python helpers (dfuse-pack.py, ...)
├── lib/main/              ← vendor SDKs and libraries (mostly submodules)
├── tools/                 ← downloaded toolchain (gcc-arm-none-eabi)
├── obj/                   ← build output (created by make)
└── downloads/             ← cached toolchain tarballs
```

## `src/main/` — the firmware

The full directory tree, grouped by layer (see [[01-overview]] for the layer model):

| Directory | Files | Role | Page |
|-----------|-------|------|------|
| `main.c`, `platform.h`, `ctype.h` | 3 | Entry point + thin shims | [[04-boot-and-scheduler]] |
| `build/` | ~10 | `version.h`, build flags, debug build markers | [[03-build-system]] |
| `common/` | ~50 | Filters, maths, encoding, string utils, color, time | — |
| `scheduler/` | 2 (`.c`+`.h`) | Cooperative real-time dispatcher | [[04-boot-and-scheduler]] |
| `fc/` | 30 | Boot, task table, RC, modes, arming, runtime config | [[05-flight-core-loop]] |
| `flight/` | ~50 | PID, mixer, IMU, GPS rescue, alt/pos hold, autopilot | [[06-flight-modules]] |
| `sensors/` | 17 | gyro/acc/baro/compass/current/voltage/esc_sensor fusion | [[07-hal-and-drivers]] |
| `drivers/` | 100+ | Abstract HAL (function-pointer driver interfaces) | [[07-hal-and-drivers]] |
| `io/` | 80+ | Serial dispatcher, VTX, GPS, LED, beeper, displayport, gimbal | [[08-io-subsystems]] |
| `msp/` | 11 | MSP wire protocol dispatcher (talks to Configurator) | [[09-msp-cli-cms]] |
| `cli/` | 4 | CLI terminal + giant settings table | [[09-msp-cli-cms]] |
| `cms/` | 20 | On-screen menu system (RC-stick navigation) | [[09-msp-cli-cms]] |
| `osd/` | 5 | OSD element rendering, warnings, stats screen | [[10-osd-blackbox-telemetry]] |
| `blackbox/` | 8 | Flight data logger (flash/SD/serial backends) | [[10-osd-blackbox-telemetry]] |
| `telemetry/` | 22 | Downlink telemetry (CRSF, SmartPort, GHST, iBus, …) | [[10-osd-blackbox-telemetry]] |
| `rx/` | 70+ | Receiver protocols: serial + SPI (CC2500, CYRF, A7105, NRF24, SX1280) | [[11-rx-subsystem]] |
| `pg/` | 80+ | Parameter Group definitions (one `.h` per config struct) | [[12-config-and-pg]] |
| `config/` | 6 | Config load/save orchestration, EEPROM streamer, feature flags | [[12-config-and-pg]] |
| `target/` | few | Generic target macros shared by all boards | [[03-build-system]] |

### `flight/` files (complete)

```
imu.c                pid.c                pid_init.c
mixer.c              mixer_init.c         mixer_tricopter.c
servos.c             servos_tricopter.c
alt_hold.h           alt_hold_multirotor.c    alt_hold_wing.c
pos_hold.h           pos_hold_multirotor.c    pos_hold_wing.c
position.c           position_estimator.c     position_filter.c   position_nav.c
gps_rescue.h         gps_rescue_multirotor.c  gps_rescue_wing.c
autopilot.h          autopilot_multirotor.c   autopilot_wing.c
flight_plan_nav.c
failsafe.c
dyn_notch_filter.c   rpm_filter.c
```

Notice the **multirotor vs wing split** — Betaflight master ships parallel implementations of every navigation feature for fixed-wing aircraft. The header (e.g. `alt_hold.h`) is the public interface; the `_multirotor.c` and `_wing.c` files implement it. The wing files were merged in 2025 as part of the "Betaflight wings" effort.

### `fc/` files (complete)

```
core.c              ← the four subtasks (RC/PID/Motor/Subprocesses)
init.c              ← three-phase boot
tasks.c             ← the task table (DEFINE_TASK array)
runtime_config.c    ← armingFlags / flightModeFlags / stateFlags
rc.c                ← rcData → rcCommand → setpoint pipeline
rc_controls.c       ← stick arm/disarm, throttle protection
rc_modes.c          ← BOX modes, mode activation conditions
rc_adjustments.c    ← real-time PID tuning via RC (sliders)
controlrate_profile.c   ← rate (deg/s) curves per profile
dispatch.c          ← deferred-callback queue (e.g. delayed disarm)
stats.c             ← arming/disarming statistics
board_info.c        ← target metadata
faults.c            ← hardfault / NMI reporting
gps_lap_timer.c     ← optional GPS lap timer
parameter_names.h   ← string table of every setting name
```

### `drivers/` top-level files

(Subdirectories `accgyro/`, `barometer/`, `compass/`, `flash/`, `rx/`, `can/`, `lcd_panel/`, `opticalflow/`, `rangefinder/` hold sensor-family-specific drivers; the top level holds bus/system/utility drivers.)

```
adc           audio.h         bus.h         bus_i2c*       bus_octospi*
bus_quadspi*  bus_spi*        buttons       camera_control  display*
dma*          dshot*          exti.h        fb_osd_impl.h   flash/
gyro_clkin.h  inverter        io*           lcd_console
lcd_panel/    light_led       light_ws2811strip
max7456       mco.h           memprot.h     motor*       nvic.h
osd*          persistent*     pin_pull_up_down  pinio
pwm_output*   resource.h      sdcard*       sdio*       sensor.h
serial*       servo_impl.h    sound_beeper  stack_check
system.h      time*           timer*        transponder_ir*
usb_io.h      usb_msc.h       vtx_*
```

### `io/` files (complete — Layer 4)

```
beeper / pidaudio                ← auditory output
dashboard                         ← legacy I²C dashboard display
displayport_max7456 / _oled / _msp / _crsf / _hott / _frsky_osd / _srxl / _fb_osd
flashfs                           ← flash file system (blackbox backend)
asyncfatfs/                       ← async FAT for SD card
gimbal / gimbal_control           ← 3-axis gimbal output
gps / gps_virtual                 ← NMEA + UBX GPS protocol parser
ledstrip                          ← WS2811/SK6812 animations
piniobox                          ← programmable I/O channels (aux outputs)
rcdevice / rcdevice_cam           ← RunCam protocol (camera control over UART)
serial / serial_resource          ← top-level serial port dispatcher
serial_4way / *_avrootloader / *_stk500v2  ← BLHeli ESC passthrough
smartaudio_protocol / tramp_protocol       ← VTX protocols (helpers)
spektrum_rssi / spektrum_vtx_control       ← Spektrum-specific
statusindicator                   ← onboard LED status
transponder_ir                    ← IR transponder for FPV racing
usb_cdc_hid / usb_msc            ← USB classes
vtx / vtx_control / vtx_msp / vtx_rtc6705 / vtx_smartaudio / vtx_tramp / vtx_table
dronecan/                         ← DroneCAN node implementation
```

### `rx/` files

```
Serial RX protocols:  sbus  spektrum  crsf  ibus  fport  ghst  srxl2  sumd  sumh
                      xbus  jetiexbus  mavlink  targetcustomserial.h
SPI RX:               cc2500_frsky_d/x/sfhss/redpine/shared/common
                      cyrf6936_spektrum   a7105_flysky
                      nrf24_v202/syma/cx10/h8_3d/inav/kn
                      expresslrs (+ _common, _impl, _telemetry)
Dispatchers:          rx.c (top-level), rx_spi.c, rx_spi_common.c
Helpers:              rx_bind.c, rc_stats.c, frsky_crc.c, pwm.c (legacy)
MSP / overrides:      msp.c, msp_override.c
```

Detailed coverage in [[11-rx-subsystem]].

### `sensors/` files (complete)

```
gyro.c / gyro_init.c / gyro_filter_impl.c   ← gyro fusion + filtering
acceleration.c / acceleration_init.c        ← accelerometer fusion
barometer.c                                  ← altitude estimation
compass.c                                    ← magnetometer + heading
battery.c                                    ← voltage + current + mAh tracking
voltage.c / current.c / current_ids.h / voltage_ids.h  ← meter abstractions
adcinternal.c                                ← MCU temp + VREFINT
esc_sensor.c                                 ← DShot bidirectional telemetry
opticalflow.c                                ← optical-flow sensors
rangefinder.c                                ← ToF + ultrasonic
boardalignment.c                             ← board-to-body rotation matrix
initialisation.c                             ← top-level sensor discovery
sensors.c / sensors.h                        ← enabledSensors bitmask
```

## `src/platform/` — MCU-family code

```
STM32/        ← STMicroelectronics (F4, F7, G4, H5, H7, C5, N6 subfamilies)
AT32/         ← Artery Semiconductor (STM32 clone, drop-in)
APM32/        ← Geehy Semiconductor (STM32 clone)
ESP32/        ← Espressif WiFi SoC
PICO/         ← Raspberry Pi RP2040
SIMULATOR/    ← SITL (native x86-64 build)
common/       ← shared STM32 subfamily helpers
```

Inside each platform (taking `STM32/` as the canonical example):

```
src/platform/STM32/
├── adc_stm32f4xx.c, ..._h7xx.c, ...        ← ADC drivers per subfamily
├── bus_i2c_ll.c, bus_spi_hal2.c, ...       ← bus drivers (HAL/LL variants)
├── bus_quadspi_hal.c, bus_octospi_stm32h7xx.c
├── can_stm32f4xx.c, ...
├── dma_stm32h7xx.c, dma_reqmap_mcu.c
├── dshot_bitbang_ll.c, dshot_bitbang_stdperiph.c
├── exti.c, gyro_clkin_stm32.c, io_stm32.c
├── light_ws2811strip_hal.c, ...
├── memprot_stm32f4xx.c, ...
├── persistent_hal.c, persistent_stdperiph.c
├── pwm_output_dshot_hal.c, pwm_output_dshot_ll.c
├── rcc_stm32f4xx.c, ...
├── sdio_f7xx.c, sdmmc_sdio_h7xx.c, ...
├── serial_uart_stm32h7xx.c, serial_usb_vcp.c, ...
├── timer_hal.c, timer_ll.c, ...
├── transponder_ir_io_*.c
├── usbd_msc_desc.c, usb_msc_f7xx.c, ...
│
├── include/                ← STM32-only headers
├── link/                   ← linker scripts (stm32_flash_f405.ld etc.)
├── mk/                     ← per-subfamily Makefile fragments
├── startup/                ← ASM startup files
├── target/                 ← per-MCU target definitions
│   ├── STM32F405/target.h target.mk
│   ├── STM32H743/target.h target.mk
│   └── ...
├── vcp/  vcpf4/  vcp_hal/  ← three generations of USB VCP stack
```

The pattern repeats for AT32, APM32, ESP32, PICO with vendor-specific naming. SIMULATOR is much smaller — it provides virtual sensor drivers (`accgyro_virtual.c`, `barometer_virtual.c`, …), TCP-based serial (`serial_tcp.c`), and a UDP link to flight simulators (`udplink.c`).

See [[07-hal-and-drivers]] for how the abstract `drivers/` and concrete `src/platform/*/` halves are bolted together at link time.

## `lib/main/` — vendor SDKs and libraries

Most are git submodules. Some hydrate automatically (`src/config`, `dronecan/libcanard`); others are opt-in (`update = none` in `.gitmodules`) and pulled only when their platform builds.

```
CMSIS/                              ← ARM Cortex core headers (always present)
STM32F4xx_HAL_Driver/               ← ST HAL libs (per subfamily folder)
STM32F7/  STM32G4/  STM32H7/  STM32C5/  STM32H5/  STM32N6/   ← opt-in
STM32_USB_Device_Library/           ← USB CDC/MSC/HID
APM32F4/                            ← Geehy SDK (opt-in)
AT32F43x/                           ← Artery SDK
GD32F4/ GD32H7/                     ← GigaDevice (opt-in)
X32M7/                              ← XMC (opt-in)
pico-sdk/                           ← Raspberry Pi (opt-in)
esp-idf/                            ← Espressif (opt-in)
FatFS/                              ← FAT filesystem for SD card
dronecan/libcanard/                 ← DroneCAN transport
BoschSensortec/                     ← Bosch sensor reference code
MAVLink/                            ← MAVLink protocol headers
google/  dyad/                      ← misc utility libs (dyad = async net for SITL)
```

## `src/config/` — board configs (separate submodule)

This is a separate git repo (`betaflight/config`) mounted as a submodule. Each subdirectory under `src/config/configs/<BOARD>/` defines one flyable board. Minimum: a single `config.h` declaring:

```c
#define FC_TARGET_MCU STM32F405
#define BOARD_NAME    MATEKF405TE
#define MANUFACTURER_ID "MTKS"
#define SYSTEM_HSE_MHZ 8
// ...feature USE_* macros and pin assignments
```

Optionally `config.c` (custom init code, only built if present), `config.mk` (extra link flags, OCTOSPI pin overrides), `defaults.txt` (CLI defaults applied at first boot).

The build resolves `make CONFIG=MATEKF405TE` to `TARGET=STM32F405` by parsing the `FC_TARGET_MCU` macro out of that config's `config.h` via the C preprocessor. See [[03-build-system]] §"Config resolution".

## `src/test/` — host-side test harness

A separate Makefile (`src/test/Makefile`) builds **non-firmware** unit tests on the host using GoogleTest. Tests are limited to pure-logic modules (PID math, mixer math, filter math, sensor scaling, MSP encoding). Real-hardware-dependent code is shimmed by fakes in `src/test/unit/`. Not part of the firmware build.

## `obj/` — build output

```
obj/
├── betaflight_2026.6.0_STM32F405.hex     ← Intel HEX, flash-ready
├── betaflight_2026.6.0_STM32F405.bin     ← raw binary
├── betaflight_2026.6.0_STM32F405.elf     ← ELF for GDB
├── betaflight_2026.6.0_STM32F405.map     ← linker memory map
├── betaflight_2026.6.0_STM32F405.lst     ← disassembly listing
└── main/<TARGET>/                         ← per-file .o and .d output
```

For EXST (external storage bootloader) targets the binary is padded and MD5-stamped; the hex is rebuilt from the patched binary. For SITL the output is a native executable.

See [[03-build-system]] §"Outputs" for the full artifact table.
