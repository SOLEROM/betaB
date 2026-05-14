---
noteId: "ab9186e04f8211f194a2c3b1eecd91b7"
tags: []

---

# Controllers

## Overview

Three Python control loops close around Betaflight via MSP-RC injection.
Each is a class in `bt_app/control/` that (a) subscribes to
`Context.on_state_changed` and only ticks while a matching `State` is
active, (b) subscribes to `Parameters.on_parameter_changed` so live gain
tuning is immediate, and (c) writes `(roll, pitch, throttle, yaw, AUX…)`
tuples through `Context.msp.set_rc(...)`. There is also a stand-alone
PID class (`pid.py`) and a `BetaflightRcMapper` for sticks↔dps conversion.

## Key Decisions

- **PID with output clamp** (`pid.py:5-87`). dt computed from
  `time.monotonic`; first tick is a no-op (`dt=0`). Output limits accept
  either symmetric scalar or `(min, max)` tuple. Anti-windup is **not**
  implemented — integral keeps growing even when saturated.
- **Three controllers, three roles.**
  - `TakeoffController` — runs in `TAKEOFF` and `LAND`. Reads
    `target_altitude_m` from `Parameters` (`takeoff_altitude`), uses
    altitude PID to set throttle, holds roll/pitch neutral, asserts
    `ARM` AUX high (`takeoff_controller.py:33-53`).
  - `HoverYawController` — runs in `SEARCH`. Keeps the altitude PID
    locked to the altitude captured on first tick (`hover_altitude`), and
    commands a constant `yaw_rate` (deg/s) converted to RC via
    `BetaflightRcMapper` (`hover_yaw_controller.py:97-134`).
  - `VisualTargetController` — drives roll/pitch/yaw/throttle from
    image-error angles. Pulls a feed-forward throttle
    `hover_throttle / (cos(roll)*cos(pitch))` so tilt doesn't bleed
    altitude (`visual_controller.py:323-336`).
- **Each controller owns its loop thread.** `HoverYawController._run`
  hits 50 Hz with monotonic clock catch-up (`hover_yaw_controller.py:78-90`);
  `TakeoffController.tick()` is called from `App.run` (no thread).
  `VisualTrackerManager.resolve` is driven by ZMQ callback (push, not pull).
- **Yaw is in dps; stick range is parameter-driven.**
  `betaflight_yaw_rate_full_stick_dps` (default 67.0, declared in
  `parameter_storage_example.yaml`) must match Betaflight's actual rate
  profile so a 90 dps command produces 90 dps.
- **Parameter→field mapping is explicit.**
  `VISUAL_TRACKER_PARAMETERS` (`visual_controller.py:20-29`) translates
  parameter names to dataclass field names, so a change to a
  YAML/CLI/ZMQ-driven value updates the live controller without restart.
- **Visual yaw uses a PID; pitch/throttle use scalar gains.** Asymmetric
  on purpose: yaw needs damping, pitch is essentially open-loop forward
  with a small Y-error correction (`visual_controller.py:207-359`).

## Constraints

- Only one controller should command RC at a time. Coordination is
  via `enable = state in (...)` checks; if two enabled controllers
  overlap, the MSP dispatcher's `set_rc` will alternate at 50 Hz and
  fight.
- Throttle output of altitude PID is added to `RC_MID (1500)`, then
  clamped to `[1000, 2000]` — meaning at neutral PID output the motors
  run at mid-throttle, not idle. Hover throttle of ~1500 is implied.
- All controllers require `Parameters` to have certain keys present
  (`altitude.kp/ki/kd/output_limits`, `takeoff_altitude`, `hover_yaw.yaw_rate`,
  `visual.*`) — listed in `parameter_storage_example.yaml`. Missing keys
  raise on construction.
- `BetaflightRcMapper` clamps to `[RC_MIN, RC_MAX]` and validates that
  `yaw_rate_full_stick_dps > 0` (`rc_mapper.py:19-39`).
- The visual controller's `pitch_visual_deg` is added to a base
  `forward_pitch_deg` (negative = nose-down = forward). Sign flips
  silently change directionality.

## Open Questions

- No anti-windup: takeoff PID will integrate to extreme values if it
  saturates throttle for long. Behavior under sustained ground-effect
  stickiness unverified.
- `HoverYawController.initialize` captures altitude on the **first
  enabled tick**, but `state` is set before `last_altitude` is
  guaranteed populated — race possible on cold start.
- Visual controller's `roll_deg` is forced to `0.0`. Why no lateral
  centering? Implied: yaw-based centering only (single forward camera).
- `LAND` state reuses `TakeoffController` with `takeoff_altitude=0.0`
  (`flight_tree.Land:148`). Hack flagged in the source itself.
- Sign-convention table (`rc_roll_sign`, `rc_pitch_sign`, `rc_yaw_sign`)
  is `ControllerConfig` defaults but never parameterised — if Betaflight
  rates change handedness, this silently breaks.
