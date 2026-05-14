---
type: architecture
status: stable
created: 2026-05-14
updated: 2026-05-14
tags: [overview, architecture, layers]
source-commit: 6434dd725
---

# 01 — Top-View Architecture

Betaflight is a hard-real-time embedded application running cooperatively on a single Cortex-M core (with a few dual-core experiments). Every line of code lives in one of **five layers**, and the lower three are gated by a **single timing master** — the gyro interrupt at 8 kHz (typical). Internalise these two ideas and the rest of the codebase becomes navigable.

## The five layers

```
┌──────────────────────────────────────────────────────────────────────┐
│ 5. Presentation / Off-Vehicle               Configurator, BB Explorer │
│                                                  msp/, cli/, cms/    │
│                                                  blackbox/, telemetry/│
│                                                  osd/                 │
├──────────────────────────────────────────────────────────────────────┤
│ 4. I/O & Features                            io/   (serial dispatcher,│
│                                              VTX, GPS, LED, beeper,   │
│                                              displayport_*, gimbal)   │
├──────────────────────────────────────────────────────────────────────┤
│ 3. Flight Core (the actual flight controller)                         │
│      fc/      task table, RC, modes, arming, init, runtime config    │
│      flight/  PID, mixer, IMU, GPS-rescue, alt/pos hold, failsafe    │
│      sensors/ gyro/acc/baro/mag/current/voltage fusion & calibration │
│      scheduler/ cooperative dispatcher with REALTIME bypass           │
├──────────────────────────────────────────────────────────────────────┤
│ 2. HAL — abstract                            drivers/                 │
│      accgyro/ baro/ compass/ motor/dshot serial bus_spi bus_i2c       │
│      timer dma flash sdcard max7456 ledstrip beeper rx/ (SPI rx)     │
├──────────────────────────────────────────────────────────────────────┤
│ 1. Platform — concrete                       src/platform/<arch>/     │
│      STM32  AT32  APM32  ESP32  PICO  SIMULATOR  common/              │
│      MCU vendor HAL, startup, linker scripts, USB stacks, VCP        │
└──────────────────────────────────────────────────────────────────────┘
```

**Layer 1 — Platform.** One directory per silicon family under `src/platform/`. Holds vendor SDK glue (STM32 HAL/LL, ATBSP, Geehy DDL, Pico SDK), startup `.s` files, linker scripts `*.ld`, MCU-family `mk/` fragments, USB VCP stacks, and concrete drivers (e.g. `serial_uart_stm32h7xx.c`).

**Layer 2 — Abstract HAL.** `src/main/drivers/` is pure C function-pointer interfaces and opaque resource types (`dmaResource_t`, `timerResource_t`, `serialPort_t` + vtable). It knows nothing about which MCU it runs on; it just describes "a UART", "a SPI bus", "a gyro that supports `initFn` + `readFn`". Layer 1 fills in the function pointers.

**Layer 3 — Flight Core.** Where the actual flight controller lives. The four sub-domains:
- `scheduler/` — the cooperative dispatcher that drives everything.
- `fc/` — boot/init, task table, RC pipeline, mode/arming logic.
- `flight/` — PID, mixer, IMU attitude estimation, GPS rescue, alt/pos hold.
- `sensors/` — wraps `drivers/` with calibration, fusion, filtering.

**Layer 4 — I/O & Features.** `src/main/io/` is the "application layer of the firmware" — serial port dispatcher, motor command distribution, VTX control protocols, GPS protocol parsers, LED strip animations, beeper, on-screen display drivers (the rendering surface, not the elements).

**Layer 5 — Presentation / Off-Vehicle.** Things the user or another machine sees: the MSP wire protocol (Configurator), the CLI terminal, the CMS on-screen menu, the OSD elements themselves, the blackbox logger, and downlink telemetry protocols.

## The single timing master — gyro-as-heartbeat

This is the most load-bearing fact in the entire architecture. **The gyro's data-ready interrupt is the only timing source that matters for the flight loop.**

```
gyro IRQ (8 kHz, every 125 µs)
   └→ scheduler busy-waits to deadline
        ├→ TASK_GYRO     (read 6 bytes over SPI/DMA)
        ├→ TASK_FILTER   (LPF1, LPF2, notch, dyn-notch)
        └→ TASK_PID      (rcCommand → PID → mixer → motorWrite)
            ↳ all three are TASK_PRIORITY_REALTIME — they bypass
              the dynamic-priority queue
   then in remaining slack:
        └→ regular tasks (RX, telemetry, OSD, CMS, blackbox, ...)
           selected by static priority × age × deadline slack
```

Everything else — RX frames at 50–500 Hz, MSP at on-demand, OSD at 60 Hz, blackbox at 1–8 kHz, GPS at 5–10 Hz — is **scheduled in the time left over** after gyro/filter/PID complete. The scheduler ages task priority so nothing starves, but the gyro deadline is sacred.

If you remember nothing else: **`taskMainPidLoop` in `src/main/fc/core.c` runs once per gyro tick, and its four subtasks (`subTaskRcCommand`, `subTaskPidController`, `subTaskMotorUpdate`, `subTaskPidSubprocesses`) are the flight controller**. Everything else exists to feed those four functions data or carry their output off the FC.

See [[04-boot-and-scheduler]] for scheduler internals and [[05-flight-core-loop]] for the subtask chain in detail.

## Configuration is orthogonal to code — the PLATFORM / TARGET / CONFIG triaxis

A single source tree builds 200+ different firmware images. The build system models this as three orthogonal axes:

| Axis | Lives in | Example | Selected by |
|------|----------|---------|-------------|
| **PLATFORM** | `src/platform/<arch>/` | `STM32`, `AT32`, `APM32`, `ESP32`, `PICO`, `SIMULATOR` | Auto, derived from TARGET |
| **TARGET**   | `src/platform/<arch>/target/<MCU>/` | `STM32F405`, `STM32H743`, `SITL` | `make TARGET=…` or derived from CONFIG |
| **CONFIG**   | `src/config/configs/<BOARD>/` (separate submodule) | `MATEKF405TE`, `IFLIGHT_BLITZ_F722` | `make CONFIG=…` |

`make CONFIG=MATEKF405TE` expands `FC_TARGET_MCU` from that config's `config.h` and resolves `TARGET=STM32F405` automatically. The config submodule (`src/config/`) lives in a separate repository on purpose: board configs change far more often than the firmware, and they're maintained collaboratively by hardware vendors.

See [[03-build-system]] for the full Makefile flow and how the three axes are resolved.

## Runtime state — three bitmasks plus a dirty config

The entire flight-state machine boils down to four global variables in `src/main/fc/runtime_config.c`:

| Variable | Type | Purpose |
|----------|------|---------|
| `armingFlags` | `uint8_t` | `ARMED`, `WAS_EVER_ARMED`, `WAS_ARMED_WITH_PREARM` |
| `flightModeFlags` | `uint16_t` | `ANGLE_MODE`, `HORIZON_MODE`, `ALT_HOLD_MODE`, `POS_HOLD_MODE`, `GPS_RESCUE_MODE`, `FAILSAFE_MODE`, … |
| `stateFlags` | `uint8_t` | `GPS_FIX`, `GPS_FIX_HOME`, `GPS_FIX_EVER` |
| `armingDisableFlags` | `uint32_t` | 25+ reasons we cannot arm: `NO_GYRO`, `RX_FAILSAFE`, `THROTTLE`, `ANGLE`, `RUNAWAY_TAKEOFF`, `CRASH_DETECTED`, … |

Plus a global `configIsDirty` flag that says "something in a Parameter Group was modified; flush to EEPROM on next save". The PG system (Layer 5 ↔ Layer 3 plumbing) is covered in [[12-config-and-pg]].

## Data-flow diagram (one cycle, 125 µs)

```
                ┌────────────────────────────────────────────────┐
                │     PHYSICAL WORLD (the quadcopter)            │
                │   gyro chip ←→ motors ←→ pilot RC link         │
                └───┬──────────────────────────────────┬─────────┘
                    │ SPI DMA (DRDY IRQ)               ▲ DShot DMA
                    ▼                                  │
            ┌───────────────┐                  ┌───────┴───────┐
            │ TASK_GYRO     │ raw 6×int16     │ writeMotors()  │
            │ src/main/...  │─────────────────│ src/main/...   │
            │ /sensors/gyro │                  │ /flight/mixer  │
            └──────┬────────┘                  └───────▲────────┘
                   │ rates rad/s                       │ motor[0..N-1] 0–2047
                   ▼                                   │
            ┌──────────────┐    rcCommand[]    ┌──────┴───────┐
            │ TASK_FILTER  │◀───────────────── │ subTaskMotor │
            │ LPF + notch  │                   │ Update       │
            └──────┬───────┘                   └───────▲──────┘
                   │ gyroADCf[]                        │ pidData[].Sum
                   ▼                                   │
            ┌──────────────────────────────────────────┴────────┐
            │ TASK_PID  →  taskMainPidLoop()                    │
            │   subTaskRcCommand   ← rcData[] from RX (50–500Hz)│
            │   subTaskPidController                            │
            │   subTaskMotorUpdate                              │
            │   subTaskPidSubprocesses ─→ blackboxUpdate()      │
            └───────────────────────────────────────────────────┘
```

## What this codebase explicitly is *not*

- **Not an RTOS user.** No FreeRTOS, no preemption (except hardware ISRs). The scheduler is a single-threaded cooperative dispatcher. Tasks may not block.
- **Not portable C++.** Pure C with a sprinkle of compiler-specific attributes (`__attribute__((section(...)))`, `FAST_CODE`, `FAST_DATA_ZERO_INIT`).
- **Not unit-tested in the running firmware.** A separate `src/test/` GoogleTest harness exists for host-side validation of pure-logic modules.
- **Not malloc-using.** All memory is statically allocated; the linker map is the memory plan. Stack canaries via `stack_check.c`.

## Where to go next

- Want to **read** the code in order? → [[04-boot-and-scheduler]] then [[05-flight-core-loop]].
- Want to **build** it? → [[03-build-system]].
- Want to **find** something? → [[02-directory-layout]].
- Want to **change** something? → [[13-modification-guide]].
