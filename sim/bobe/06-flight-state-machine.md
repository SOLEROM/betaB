---
noteId: "9a54af604f8211f194a2c3b1eecd91b7"
tags: []

---

# Flight State Machine

## Overview

Mission orchestration lives in `bt_app/behaviors/flight_tree.py`,
implemented as a `py_trees` Sequence (memoryful) plus a shared
`Context` carrying live state and a `State` IntEnum
(`bt_app/common/__init__.py:13-21`: DISARMED, ARMED, TAKEOFF, LAND,
VISUAL_TRACK, FINAL, SEARCH). The tree itself does not run controllers
— it transitions `Context.state`, and controllers subscribe to
`on_state_changed` and enable themselves when the relevant state is
active. The tree ticks at 10 Hz (`time.sleep(0.1)` in `main.run`).

## Key Decisions

- **Sequence with memory.** A failing child fails the whole sequence; a
  succeeding child advances. Memory means a phase that succeeded once
  isn't re-evaluated (`flight_tree.py:225`).
- **Six phases, hardcoded.**
  1. `Disarmed Neutral` — 2 s of disarmed-throttle RC (`TimedRcCommand`).
  2. `Arm Low Throttle` — 2 s of armed-throttle RC.
  3. `TakeoffUntilAltitude` — sets `State.TAKEOFF` and watches blackboard
     `/flight/current_altitude_m` ≥ `target_altitude_m`.
  4. `Search` — sets `State.SEARCH` (yaws in place via `HoverYawController`).
  5. `VisualTrack` — sets `State.VISUAL_TRACK`, blocks until lidar range
     ≤ `visual.final_tracking_distance` or `visual.tracking_timeout_s`.
  6. `FinalTrack` — sets `State.FINAL` (RUNNING forever; demo end).
- **Blackboard as RW bus.** `pre_tick_handler` writes
  current/target altitude and lidar range to the py_trees blackboard
  every tick so behaviors can read without poking into `Context`
  (`main.py:95-99`).
- **Hardcoded RC tuples.** Arming sequence uses fixed 8-channel tuples
  (`flight_tree.py:15-18`) — not parameterised. Throttle channel index
  hardcoded to 2.
- **`WaitArmable` exists but is unused in `create_app_tree`.** It
  defines the "wait for FC to clear blocking arming-disable flags" logic;
  the live tree currently just blasts the arm RC for 2 s.
- **State transitions are imperative side-effects.** Each behavior's
  `initialise()` calls `ctx.set_state(...)`, which fires `on_state_changed`
  and enables the matching controller.

## Constraints

- The tree ticks at 10 Hz; controllers tick at 50 Hz independently — the
  tree is a slow phase-selector, not a control loop.
- Behaviors are stateless across re-init; failing a behavior re-enters
  `initialise()` next time around (py_trees default).
- `BB_IS_RUNNING_KEY` is declared in the writer blackboard but never
  read by any behavior — appears unused.
- `Search` and `FinalTrack` return `RUNNING` forever; only external
  state changes can advance them.

## Open Questions

- No abort branch. There's no "if anything goes wrong, transition to
  LAND" subtree. A failed phase just hangs the sequence at `RUNNING`/
  `FAILURE` with no recovery.
- `WaitArmable` is dead code — drop or wire it before `Arm Low Throttle`?
- Lidar-based "final tracking distance" assumes the front lidar is
  pointing at the target; no validation.
- Mission completion: `FinalTrack.update` always returns RUNNING. There
  is no "mission done" event, so `app.run` only exits on Ctrl-C.
- The state machine is open-loop with respect to arming actually
  happening — we set RC and assume Betaflight armed.
