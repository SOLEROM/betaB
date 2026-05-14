---
type: architecture
status: stable
created: 2026-05-14
updated: 2026-05-14
tags: [flight, modules, pid, mixer, imu, navigation]
source-commit: 6434dd725
---

# 06 — Flight Modules Inventory

This is a one-line-per-file catalogue of every source file under `src/main/flight/` and `src/main/fc/`. The headline functions are picked from each so you can `grep` straight to the right symbol.

## `src/main/flight/` — feature implementations

### Core control loop

| File | Role | Key symbols |
|------|------|-------------|
| `pid.c`, `pid.h` | PID controller for ROLL/PITCH/YAW. P, I, D, F, S terms. iTerm relax, anti-gravity, TPA, abs-control, crash recovery, runaway protection. | `pidController()`, `pidData[3]`, `pidRuntime` |
| `pid_init.c`, `pid_init.h` | PID profile activation. Recomputes filter coefficients, scales gains by PTERM_SCALE/ITERM_SCALE/DTERM_SCALE, primes feed-forward. | `pidInit()`, `pidInitFilters()` |
| `mixer.c`, `mixer.h` | Motor mixing matrix. Combines throttle + pidSum into per-motor commands. Airmode, dynamic idle, thrust linearisation. | `mixTable()`, `motor[]`, `motor_disarmed[]` |
| `mixer_init.c`, `mixer_init.h` | Mixer mode tables (QUADX, QUADP, HEX, OCTOX8, TRI, BICOPTER, FLYING_WING, AIRPLANE, …). Custom mixer loader. | `mixerInit()`, `mixerConfigureOutput()` |
| `mixer_tricopter.c`, `mixer_tricopter.h` | Tricopter-specific servo mixing (tail yaw servo + 3 motors). | `mixerTricopterInit()`, `mixerTricopterUpdate()` |
| `servos.c`, `servos.h` | Servo PWM output for non-multirotor mixers. | `servoMixer()`, `writeServos()` |
| `servos_tricopter.c` | Tricopter tail-servo specifics, including yaw-feedback from servo angle. | `tricopterServoUpdate()` |
| `imu.c`, `imu.h` | Attitude estimation. DCM rotation matrix + complementary filter using gyro + accel + (optional) compass. Outputs Euler angles and quaternion. | `imuUpdateAttitude()`, `attitude`, `rMat`, `imuAttitudeQuaternion` |

### Filtering

| File | Role |
|------|------|
| `dyn_notch_filter.c`, `dyn_notch_filter.h` | Dynamic notch filter. FFT over gyro data, tracks dominant motor frequencies, applies up to N notches per axis. |
| `rpm_filter.c`, `rpm_filter.h` | RPM-driven notch filter. Uses bidirectional DShot ESC telemetry to place notches exactly at each motor's blade-pass frequency. |

### Altitude / position / navigation (multirotor + wing variants)

Each feature has a shared header plus split implementation files for multirotor and fixed-wing aircraft. The split is recent — Betaflight master merged the "Betaflight Wings" effort that brings fixed-wing support back from a fork.

| Header | Multirotor | Wing | Purpose |
|--------|------------|------|---------|
| `alt_hold.h`     | `alt_hold_multirotor.c` | `alt_hold_wing.c` | Barometer-based altitude hold. Climb/descend rate via stick deflection. |
| `pos_hold.h`     | `pos_hold_multirotor.c` | `pos_hold_wing.c` | GPS-based position hold. Pilot can nudge. |
| `gps_rescue.h`   | `gps_rescue_multirotor.c` | `gps_rescue_wing.c` | Return-to-home on failsafe or pilot command. Climb-cruise-descend-land state machine. |
| `autopilot.h`    | `autopilot_multirotor.c` | `autopilot_wing.c` | Waypoint navigation framework. |

Plus shared position infrastructure:

| File | Role |
|------|------|
| `position.c`, `position.h` | Top-level position estimator. Selects which source feeds altitude (baro / GPS / fusion). |
| `position_estimator.c`, `position_estimator.h` | Multi-sensor fusion (complementary filter, optionally EKF-style). |
| `position_filter.c`, `position_filter.h` | Per-axis filtering for the estimator's inputs/outputs. |
| `position_nav.c`, `position_nav.h` | Navigation controller — turns waypoints into roll/pitch setpoints. |
| `flight_plan_nav.c`, `flight_plan_nav.h` | Programmable flight plans (waypoint sequences). |

### Safety

| File | Role | Key symbols |
|------|------|-------------|
| `failsafe.c`, `failsafe.h` | RC loss recovery state machine. STAGE1 (hold), STAGE2 (drop/land/RTH). Configurable stick failsafe and box failsafe. | `failsafeUpdateState()`, `failsafePhase` |

## `src/main/fc/` — flight-controller orchestration

| File | Role | Key symbols |
|------|------|-------------|
| `core.c`, `core.h` | The four PID-loop subtasks. Arm/disarm. Runaway-takeoff. Crash detection. | `taskMainPidLoop()`, `subTask*()`, `tryArm()`, `disarm()` |
| `init.c`, `init.h` | 3-phase boot (`initPhase1/2/3`). Probes hardware, loads config, primes scheduler. | `initPhase1()`, `initPhase2()`, `initPhase3()` |
| `tasks.c`, `tasks.h` | The task table. `DEFINE_TASK` entries for ~40 tasks. `tasksInit()` enables them per feature flags. | `tasks[]`, `setTaskEnabled()` |
| `runtime_config.c`, `runtime_config.h` | `armingFlags`, `flightModeFlags`, `stateFlags`, `armingDisableFlags`. The flight-state bitmasks. | `enableFlightMode()`, `setArmingDisabled()` |
| `rc.c`, `rc.h` | RC processing pipeline. Smoothing, expo, rates, deadband, setpoint output. | `processRcCommand()`, `getSetpointRate()`, `getRcDeflection()` |
| `rc_controls.c`, `rc_controls.h` | Stick-position interpretation. Arm/disarm by sticks, calibration triggers. | `processRcStickPositions()`, `isUsingSticksForArming()` |
| `rc_modes.c`, `rc_modes.h` | BOX activation logic. Maps AUX channel ranges to `flightModeFlags`. | `updateActivatedModes()`, `IS_RC_MODE_ACTIVE()` |
| `rc_adjustments.c`, `rc_adjustments.h` | Real-time tuning via RC (slider-style PID/rate tweaks during flight). | `processRcAdjustments()` |
| `controlrate_profile.c`, `controlrate_profile.h` | Rate profile management. `controlRateConfig_t` holds one rate set. | `changeControlRateProfile()` |
| `dispatch.c`, `dispatch.h` | Deferred-callback queue. Schedule one-shots N µs in the future (used by delayed disarm, beeper sequences, etc.). | `dispatchAdd()`, `dispatchProcess()` |
| `stats.c`, `stats.h` | Arming/flight statistics persisted to EEPROM. Cumulative flight time, arms count, etc. | `statsOnArm()`, `statsOnDisarm()` |
| `board_info.c`, `board_info.h` | Board metadata (manufacturer, board name). Burned into the image at build time. | `boardInformationGetBoardName()` |
| `faults.c`, `faults.h` | Hardfault/NMI reporting. Optional persistent fault log. | `faultsHardFault()`, `faultsCheckFlash()` |
| `gps_lap_timer.c`, `gps_lap_timer.h` | Optional GPS-based lap timer for racing. | `gpsLapTimerProcess()` |
| `parameter_names.h` | String table mapping every CLI setting name to a numeric ID. Used by MSP_GET_SETTING_INFO and CLI completion. | `lookupTableName[]` |

## How they wire together

Picture the dependency graph in three rings:

```
Outer ring (off-vehicle / I/O):
   msp/  cli/  cms/  osd/  blackbox/  telemetry/  rx/  io/  drivers/  src/platform/

Middle ring (sensors + features):
   sensors/                  flight/
   ├ gyro          ─────────► imu ──► attitude
   ├ acc           ─────────►       ─► angle/horizon modes
   ├ baro          ─────────► position ─► alt_hold
   ├ compass       ─────────►          ─► magHold (in subTaskPidSubprocesses)
   ├ battery       ─────────► (mixer voltage compensation)
   └ esc_sensor    ─────────► rpm_filter ──┐
                                            │
   GPS (io/gps.c) ───────────► position    │
                              ─► gps_rescue│
                              ─► pos_hold  │
                              ─► autopilot │
                                            ▼
Inner ring (the loop):                  flight/
   fc/core.c::taskMainPidLoop                    
   ├ subTaskRcCommand    ◄── fc/rc.c ◄── rx/ (rcData[])
   ├ subTaskPidController ── flight/pid.c ◄── filters ◄── gyro
   ├ subTaskMotorUpdate  ── flight/mixer.c ──► drivers/motor.c ──► ESCs
   └ subTaskPidSubprocesses ─► blackbox/, magHold update
```

`fc/rc_modes.c` injects mode flags that change setpoint computation in `flight/pid.c` (Angle, Horizon, Headfree) and shifts the loop to a different controller entirely for GPS Rescue / Position Hold / Autopilot.

## A few non-obvious things

### Speed-optimised files

The build (`mk/source.mk`) flags these as `SPEED_OPTIMISED_SRC` (compiled with `-Ofast` + LTO):

```
common/filter.c     flight/imu.c        flight/pid.c
flight/mixer.c      sensors/gyro.c      sensors/gyro_filter_impl.c
flight/dyn_notch_filter.c
```

If you touch one of these and the build slows down, that's expected — they get aggressive optimisation.

### `FAST_CODE` and friends

Hot functions are annotated `FAST_CODE` or `FAST_CODE_NOINLINE` so the linker places them in tightly-coupled memory (CCM/ITCM on STM32). On chips with such memory, `taskGyroSample`, `taskFiltering`, `taskMainPidLoop`, `pidController`, `mixTable`, `imuUpdateAttitude` and their callees live there. `FAST_DATA_ZERO_INIT` puts hot data structures in DTCM. Look for these macros to find what the linker treats as latency-critical.

### Profile vs config

Three things called "profile" in Betaflight:

| Profile | Stored in | Active index |
|---------|-----------|--------------|
| PID profile (gains, filter cutoffs, anti-gravity, …) | `pidProfile_t[PID_PROFILE_COUNT]` (default 4) | `systemConfig()->pidProfileIndex` |
| Rate profile (rates, expo, throttle limit) | `controlRateConfig_t[CONTROL_RATE_PROFILE_COUNT]` (default 6) | `systemConfig()->activeRateProfile` |
| OSD profile (which elements visible, positions) | `osdConfig_t->item_pos[3][...]` | `osdConfig()->osdProfileIndex` |

Profiles can be switched from RC via dedicated mode boxes (`BOXPIDPROFILE_INC`, `BOXPIDPROFILE_DEC`, etc.).

### Wing vs multirotor at compile time

Wing support is gated by `USE_WING` in build flags. The split files (`*_wing.c`) compile only when that's set; the multirotor variant is the default. The `.h` header stays shared and acts as the polymorphic contract.

### IMU coordinate system

Body frame: X = forward, Y = right, Z = down (NED if you want to think aerospace). Gyro samples are in this frame post-`boardalignment.c` rotation. Euler `attitude.values.roll/pitch/yaw` are in 0.1° units (1800 = 180°).

## Files this chapter intentionally omits

- `src/main/flight/*` is the controllers; the data they consume lives in `sensors/` and `drivers/` — see [[07-hal-and-drivers]].
- I/O surfaces (servos hardware output, motor PWM/DShot generation) are in `drivers/motor*`, `drivers/pwm_output*`, `drivers/dshot*` — see [[07-hal-and-drivers]] and [[08-io-subsystems]].
- Off-vehicle interfaces (CLI, MSP, blackbox, OSD elements) are in their own folders — see [[09-msp-cli-cms]] and [[10-osd-blackbox-telemetry]].
