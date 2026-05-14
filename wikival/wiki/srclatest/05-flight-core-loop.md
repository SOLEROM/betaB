---
type: architecture
status: stable
created: 2026-05-14
updated: 2026-05-14
tags: [pid, mixer, rc, modes, arming, flight-loop]
source-commit: 6434dd725
---

# 05 — Flight Core Loop

The flight controller, the actual quadcopter brain, is implemented in four functions called sequentially every PID cycle from `taskMainPidLoop()` in **`src/main/fc/core.c`**:

```c
FAST_CODE void taskMainPidLoop(timeUs_t currentTimeUs) {
    subTaskRcCommand(currentTimeUs);       // RC sticks → setpoint
    subTaskPidController(currentTimeUs);   // setpoint + gyro → pidData[].Sum
    subTaskMotorUpdate(currentTimeUs);     // pidData → motor[] → ESCs
    subTaskPidSubprocesses(currentTimeUs); // blackbox, magHold, runaway-takeoff
}
```

This page walks each subtask, then the RC chain that feeds it, then modes and arming.

## subTaskRcCommand — RC sticks → setpoint

`src/main/fc/core.c:1346-1367`. Calls `processRcCommand()` from `src/main/fc/rc.c`.

```
RX frame              rcData[]           rcCommand[]            setpoint[]
(50–500 Hz)           1000..2000 µs      ±500 normalised         deg/s desired
                                         (after expo + rates)    rotation rate
```

### Inputs

`extern int16_t rcData[MAX_SUPPORTED_RC_CHANNEL_COUNT];` — populated by the RX driver. Channel order:

| Idx | Function |
|-----|----------|
| 0   | ROLL     |
| 1   | PITCH    |
| 2   | YAW      |
| 3   | THROTTLE |
| 4+  | AUX1, AUX2, …                |

If the transmitter sends in a different order, `MSP_RX_MAP` / CLI `rcmap` reorders.

### Transforms (`processRcCommand` chain)

1. **Smoothing.** If `rc_smoothing_type = INTERPOLATION/FILTER`, a PT1/PT2 filter is applied to bridge the gap between RC frames (which arrive at ~33–500 Hz) and the PID loop (8 kHz). The smoothed value is used downstream.
2. **Deadband.** Roll/pitch/yaw centred values within `rc_deadband` are zeroed; throttle deadband is `yaw_deadband`.
3. **Expo curve.** `roll_expo`, `pitch_expo`, `yaw_expo` apply an exponential softening near centre.
4. **Rates.** `roll/pitch/yaw_rate` plus `rc_rate` and `super_rate` map stick deflection to target deg/s. Multiple "rate types" supported: BETAFLIGHT, RACEFLIGHT, KISS, ACTUAL, QUICK.
5. **Setpoint output.** `getSetpointRate(axis)` returns the desired body-frame rotation rate in deg/s, `getRcDeflection(axis)` returns the stick position in [-1, +1] (used elsewhere by feed-forward, anti-gravity, iTerm-relax).

`rcCommand[]` (range ±500) is the intermediate value after expo but before rates; it's also what flight mode controllers consume when overriding pilot input (angle mode treats roll/pitch `rcCommand` as a target lean angle in degrees).

### Mode-dependent setpoint

- **Acro / Air mode** — setpoint = stick-derived deg/s, no levelling.
- **Angle mode** — setpoint = `angle_gain × (target_angle − measured_angle)`. Target angle from `rcCommand`, measured from `attitude.values`.
- **Horizon mode** — blend between angle and acro based on stick deflection (more stick → more acro).
- **GPS rescue / Failsafe / Position hold** — flight feature overrides setpoint to its own commanded rate (heading hold, return-to-home steering).

## subTaskPidController — the PID itself

`src/main/fc/core.c:1217-1269` → calls `pidController(currentPidProfile, currentTimeUs)` in `src/main/flight/pid.c`.

```
error  = setpoint − gyroRate         (in deg/s)
P      = error × Kp                  (scaled by PTERM_SCALE = 0.032)
I     += error × dt × Ki             (anti-windup, iTerm relax, throttle reset)
D      = − Δ(gyroRate)/dt × Kd       (gyro-based derivative, not error-based)
F      =   Δ(setpoint)/dt × Kf       (feed-forward)
S      = (1−setpoint_weight) × error (setpoint weighting reduces P overshoot)
Sum    = clamp(P+I+D+F+S, ±pidSumLimit)
```

The result lives in **`extern pidAxisData_t pidData[3]`** — one entry per axis (ROLL, PITCH, YAW) with `.P .I .D .F .S .Sum` fields. `.Sum` is what the mixer consumes.

### Profile system

Three PID **profiles** are stored in flash; the active one is selected by `systemConfig()->pidProfileIndex`. A profile holds all gain values plus filter cutoffs, feedforward configuration, iTerm relax thresholds, anti-gravity gain, dyn-LPF settings, throttle TPA, etc. Defined in `src/main/pg/pid.h`.

Profile switching can be triggered by an RC aux channel (`pidProfileChange` mode condition).

### Advanced features layered on top

- **TPA (Throttle PID Attenuation).** Reduce P+D at high throttle to prevent vibration coupling.
- **Anti-gravity.** Boost I-term during fast throttle changes to prevent pitch-down on punch-out / pitch-up on chop.
- **iTerm relax.** Suppress integral when the pilot's stick is moving fast (flips/rolls).
- **Crash recovery.** Detect crashes (gyro + D spikes) and zero/decay terms to avoid prop-strikes.
- **Runaway takeoff protection.** If `pidSum > 600` and gyro > threshold for 75 ms, disarm — implemented inline in `subTaskPidController`.
- **Absolute control (`abs_control`).** Add a slow correction that cancels cumulative attitude error in acro.
- **Feed-forward averaging.** 2/3/4-point moving average to soften the feed-forward derivative.
- **Thrust linearisation.** Compensate for non-linear motor thrust around mid-throttle.

All of these live in `src/main/flight/pid.c`, gated by `pidRuntime.*` flags initialised by `pidInit()` in `src/main/flight/pid_init.c`.

## subTaskMotorUpdate — mixer + motor output

`src/main/fc/core.c:1309-1344`:

```c
mixTable(currentTimeUs);    // pidData[] + throttle → motor[0..N-1]
writeServos();              // servo channels for tricopter/wing
writeMotors();              // motor[] → ESC (PWM, OneShot, DShot)
```

### The mixer matrix

`motorMixer_t` in `src/main/flight/mixer.h`:

```c
typedef struct motorMixer_s {
    float throttle;   // usually 1.0
    float roll;       // ±1.0 depending on motor position
    float pitch;      // ±1.0
    float yaw;        // ±1.0 (sign reflects spin direction)
} motorMixer_t;
```

Per-motor output:

```
motor[i] = throttle              × weight[i].throttle
         + pidData[ROLL].Sum     × weight[i].roll
         + pidData[PITCH].Sum    × weight[i].pitch
         + pidData[YAW].Sum      × weight[i].yaw
```

Plus airmode constraints (lift output so the lowest-saturated motor is at min), motor-output limiting, dynamic idle, thrust linearisation, etc.

Mixer mode (`MIXER_QUADX`, `MIXER_HEXACOPTER`, …) selects which preset matrix to load — see `src/main/flight/mixer_init.c`. Custom mixers can also be programmed via CLI.

### Motor write paths

`writeMotors()` dispatches through `motor.h`'s vtable to the active motor protocol driver:

| Protocol | Driver |
|----------|--------|
| PWM (1000–2000 µs) | `drivers/pwm_output*` |
| OneShot125 / OneShot42 / MultiShot | same, different rates |
| DShot150 / 300 / 600 / 1200 | `drivers/pwm_output_dshot*` + DMA |
| DShot bidirectional (telemetry) | `dshot_bitbang*` for software decode |

Motor output is range 0–2047 (DShot) or PWM µs depending on protocol.

## subTaskPidSubprocesses — housekeeping

`src/main/fc/core.c:1271-1293`. Runs at PID rate, does:

- **`magHold` update.** If `MAG_MODE` is active, correct yaw setpoint based on compass error.
- **`blackboxUpdate()`.** Log selected fields at the configured rate (1, 2, 4, 8, … gyro cycles per logged frame).
- **`updateRcRefreshRate()`.** Track and report effective RC update rate (Configurator shows this).

## RC chain end-to-end

```
RX hardware (UART/SPI)
   └→ src/main/rx/<protocol>.c   decodes frame
        └→ rcChannels[]          raw decoded values (1000..2000 typical)
             └→ src/main/rx/rx.c rcDataProcessing(), failsafe checks
                  └→ rcData[]    sanitised, in correct channel order
                       └→ subTaskRcCommand (PID loop)
                            └→ src/main/fc/rc.c
                                  smoothing → expo → rates
                                  → rcCommand[] (±500)
                                  → setpointRate[]    (deg/s)
                                  → rcDeflection[]    (±1)
```

Failsafe is a layer between rx.c and rcData[]: if no good RX frames in `rxConfig()->rxFailsafeMs`, channels switch to user-configured failsafe values (drop, hold, set fixed).

See [[11-rx-subsystem]] for the RX protocol catalog.

## Flight modes (BOX system)

Flight modes are tracked as bits in **`flightModeFlags`** (`src/main/fc/runtime_config.h:82-96`):

```c
typedef enum {
    ANGLE_MODE      = 1 << 0,
    HORIZON_MODE    = 1 << 1,
    MAG_MODE        = 1 << 2,
    ALT_HOLD_MODE   = 1 << 3,
    POS_HOLD_MODE   = 1 << 5,
    HEADFREE_MODE   = 1 << 6,
    PASSTHRU_MODE   = 1 << 8,
    FAILSAFE_MODE   = 1 << 10,
    GPS_RESCUE_MODE = 1 << 11,
    AUTOPILOT_MODE  = 1 << 12,
} flightModeFlags_e;
```

The "box" abstraction sits between physical AUX channels and these bits. From `src/main/fc/rc_modes.h`:

```c
typedef struct modeActivationCondition_s {
    boxId_e        modeId;             // BOXANGLE, BOXALTHOLD, ...
    uint8_t        auxChannelIndex;    // AUX1..AUX_COUNT
    channelRange_t range;              // start/end (in 25-µs steps from 900)
    modeLogic_e    modeLogic;          // OR / AND with linked box
    boxId_e        linkedTo;           // chain with another condition
} modeActivationCondition_t;
```

The condition list is in `rcModeActivationConfig()` (a PG). `updateActivatedModes()` runs each RX cycle, ANDing/ORing conditions and producing `flightModeFlags`.

40+ boxes exist (`BOXARM`, `BOXAIRMODE`, `BOX3D`, `BOXCRASHFLIP`, `BOXBEEPERON`, `BOXLEDLOW`, `BOXPARALYZE`, …) — not all become `flightModeFlags` bits; some are pure latching switches read elsewhere (e.g. `BOXBEEPERON` is consumed by the beeper driver).

Mode tests in code use:
```c
FLIGHT_MODE(ANGLE_MODE | HORIZON_MODE)
IS_RC_MODE_ACTIVE(BOXAIRMODE)
```

## Arming state machine

Three orthogonal masks:

```c
extern uint8_t  armingFlags;          // ARMED | WAS_EVER_ARMED | WAS_ARMED_WITH_PREARM
extern uint16_t flightModeFlags;
extern uint8_t  stateFlags;           // GPS_FIX | GPS_FIX_HOME | GPS_FIX_EVER
static uint32_t armingDisableFlags;   // bitmask of reasons we can't arm
```

`armingDisableFlags` (`runtime_config.h:42-72`) tracks every reason arming is blocked. Sample bits:

```
ARMING_DISABLED_NO_GYRO            sensor missing
ARMING_DISABLED_FAILSAFE           failsafe active
ARMING_DISABLED_RX_FAILSAFE        no RX
ARMING_DISABLED_BAD_RX_RECOVERY    just recovered, need stable signal
ARMING_DISABLED_THROTTLE           throttle not at idle
ARMING_DISABLED_ANGLE              tilted past max_arm_angle
ARMING_DISABLED_BOOT_GRACE_TIME    < grace period since boot
ARMING_DISABLED_NOPREARM           prearm not set
ARMING_DISABLED_LOAD               CPU overloaded
ARMING_DISABLED_CALIBRATING        gyro/acc cal in progress
ARMING_DISABLED_CLI                CLI session active
ARMING_DISABLED_CMS_MENU           CMS open
ARMING_DISABLED_RUNAWAY_TAKEOFF    automatic disarm in progress
ARMING_DISABLED_CRASH_DETECTED     crash detected
ARMING_DISABLED_PARALYZE           PARALYZE mode latched
...25+ flags total
```

`disarmReason_e` (`runtime_config.h:34-40`) categorises disarm causes for blackbox / OSD: `DISARM_REASON_ARMING_DISABLED`, `DISARM_REASON_FAILSAFE`, `DISARM_REASON_THROTTLE_TIMEOUT`, `DISARM_REASON_STICKS`, `DISARM_REASON_SWITCH`, `DISARM_REASON_CRASH_PROTECTION`, `DISARM_REASON_RUNAWAY_TAKEOFF`, `DISARM_REASON_GPS_RESCUE`, `DISARM_REASON_BOOTGRACE`.

### Arming flow

1. Pilot toggles AUX channel mapped to `BOXARM`.
2. `tryArm()` in `src/main/fc/core.c` runs each cycle while `BOXARM` is active.
3. Check `isArmingDisabled()` — if any disable bit set, beep "denied" and exit.
4. Check throttle position (must be low unless `airmode` or `motor stop` exempts).
5. Set `ARMED` in `armingFlags`, set `WAS_EVER_ARMED`.
6. Reset PID state, clear iTerm, prime mixer, enable motor output.
7. Beep arm tune via `BEEPER_ARMING`.

### Disarm flow

`disarm(disarmReason_e reason)` clears `ARMED`, stops motors, records reason in stats, fires `BEEPER_DISARMING`, logs to blackbox. Conditions that auto-call `disarm()`:

- Stick disarm (throttle low + yaw extreme, if `disarm_kill_switch` not set)
- Failsafe stage 2 (configurable: LAND / GPS_RESCUE / DROP)
- Crash detection
- Runaway takeoff timer expired
- Throttle below `min_throttle_count` after arming for `throttle_timeout` (if enabled)

## Runtime config — the four bitmasks

| Mask | Where | Purpose |
|------|-------|---------|
| `armingFlags`         | `runtime_config.c` | Are we armed |
| `flightModeFlags`     | `runtime_config.c` | Which flight modes are active |
| `stateFlags`          | `runtime_config.c` | GPS state |
| `armingDisableFlags`  | `runtime_config.c` (file-static) | Why we can't arm |
| `enabledSensors`      | `sensors/sensors.c` | Which sensors detected at boot |

All flag access goes through inline helpers: `ARMING_FLAG(...)`, `FLIGHT_MODE(...)`, `STATE(...)`, `sensors(SENSOR_*)`. Set/clear via `ENABLE_*` / `DISABLE_*` or `setArmingDisabled()` / `unsetArmingDisabled()`.

## Putting it together — one PID cycle

For 8 kHz gyro, 4 kHz PID (denom=2), 100 Hz RC over CRSF:

```
t=0     gyro IRQ → SPI DMA fetches sample
t=5     taskGyroSample   raw int16 → gyroRate[XYZ]
t=15    taskFiltering    LPF1, LPF2, RPM-notch, dyn-notch → gyroADCf[]
        (only every 2nd cycle when denom=2)
t=45    taskMainPidLoop  (only every 2nd cycle when denom=2)
            subTaskRcCommand → processRcCommand → setpoint[]
                (uses rcData[] last updated by TASK_RX at ~100 Hz)
            subTaskPidController → pidData[].Sum
            subTaskMotorUpdate → mixTable → writeMotors (DShot DMA out)
            subTaskPidSubprocesses → blackboxUpdate
t=110   scheduler picks one slack task
t=120   slack task returns
t=125   next gyro IRQ
```

Inputs from outside the loop arrive asynchronously:
- TASK_RX (50–500 Hz) refreshes `rcData[]`
- TASK_ATTITUDE (100 Hz) updates `attitude.values` and the quaternion (used by Angle/Horizon)
- TASK_BARO (typically 50 Hz) feeds altitude for alt-hold
- TASK_GPS (5–10 Hz) feeds GPS for rescue/position-hold

The PID loop never blocks on any of these — it reads the latest cached value and proceeds.

## Files for this chapter

| Concern | File |
|---------|------|
| Subtask chain | `src/main/fc/core.c` (`taskMainPidLoop`, `subTask*`) |
| Gyro task | `src/main/fc/core.c:taskGyroSample` |
| Filter task | `src/main/fc/core.c:taskFiltering` |
| RC pipeline | `src/main/fc/rc.c`, `rc.h` |
| Stick arming | `src/main/fc/rc_controls.c` |
| Modes/boxes | `src/main/fc/rc_modes.c`, `rc_modes.h` |
| Real-time tuning sliders | `src/main/fc/rc_adjustments.c` |
| Runtime config flags | `src/main/fc/runtime_config.c`, `.h` |
| PID controller | `src/main/flight/pid.c`, `pid_init.c`, `pid.h` |
| Mixer | `src/main/flight/mixer.c`, `mixer_init.c` |
| IMU/attitude | `src/main/flight/imu.c`, `.h` |
| Profile storage | `src/main/pg/pid.h`, `controlrate_profile.c` |

Next: [[06-flight-modules]] catalogues every other file under `flight/`.
