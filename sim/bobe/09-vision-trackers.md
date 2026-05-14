---
noteId: "c692faa04f8211f194a2c3b1eecd91b7"
tags: []

---

# Vision Trackers

## Overview

The vision pipeline detects a tracking target in camera frames and emits
**angular errors** that the visual controller consumes. Two trackers
ship: a HSV-based `RedBoxTracker` for the demo (target is the red 1m³
box in the world) and an `OpticalFlowTracker` as an alternative. Both
run as their own processes, subscribe to the camera ZMQ topic, and
publish a `TrackerResult { error_x, error_y }` (radians) on the
`tracker_result` ZMQ topic. The data flow is one-way; the controller
only consumes results, never queries the tracker.

## Key Decisions

- **Decoupled processes communicating via ZMQ.** Tracker is a separate
  process with its own deps (cv2, numpy). `bt_app` does not import cv2
  — keeps the autopilot's startup fast and dependency surface small.
- **Errors expressed in radians, not pixels.** `RedBoxTracker` converts
  pixel error to angular error using
  `CAMERA_HORIZONTAL_FOV_RAD = π/2` (90°)
  (`red_box_tracker.py:28`). The controller multiplies by `math.degrees`
  before applying its gains (`visual_controller.py:458-461`). Angular
  representation is independent of resolution.
- **Result payload is JSON over ZMQ PUB/SUB.**
  `{"error_x": float, "error_y": float}` on topic `b"tracker_result"`
  at `ipc:///tmp/bt_app.tracker_result` (`bt_app/common/__init__.py:10-11`).
- **`RedBoxDetection` is the rich internal type** (center px, bbox,
  area, errors in both px / rad / deg), but only the `TrackerResult`
  subset crosses the wire.
- **Display is optional.** The tracker can open a CV2 window for
  debugging (`display=True`) — fine inside a dev container with X11
  forwarding, never in headless production.
- **Min-area filter.** `min_area_px=150.0` discards spurious red pixels
  (`red_box_tracker.py:109`) — implicit assumption about target distance.

## Constraints

- The camera FOV constant is hardcoded; if the iris model's camera FOV
  changes in the SDF, this constant must be updated in lockstep.
- HSV thresholds for "red" are baked into the tracker (in code, not
  parameters) — a different target color requires code edits.
- Subscriber-side: only the **latest** result matters; older messages
  must be drained (the visual controller's `_receive_loop` does this).
- Publisher must `bind` before subscriber `connect` for ZMQ PUB/SUB
  to deliver early messages (same as sensor bridges).

## Open Questions

- Tracker lifecycle: who starts and supervises the tracker process?
  No script in `bt_bringup/launch/` starts it. Manual today.
- `optical_flow_tracker.py` is a parallel implementation — when do
  we pick it over the red-box version? No selection mechanism.
- `red_box_tracker_ex.py` is a third variant — purpose vs. base?
- `target_visible` is always `True` in `VisualTrackerManager.resolve`
  (`visual_controller.py:457`) — the tracker doesn't currently send
  a "no detection" message, so the controller can't enter the
  reset/hold-throttle branch. Add a "no target" signal?
- No anti-jitter / temporal smoothing on the error stream — relies on
  deadband + PID damping.
