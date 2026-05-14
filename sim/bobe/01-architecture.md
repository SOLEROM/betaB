---
noteId: "4a1ea0a04f8211f194a2c3b1eecd91b7"
tags: []

---

# Architecture

## Overview

The system is a **multi-process pipeline** that flies a simulated quadrotor
in Gazebo using real Betaflight firmware (compiled as SITL) under the
command of a Python autopilot. Five processes run side-by-side and
communicate over a mix of UDP (sim ↔ FC), TCP/WebSocket (Configurator ↔ FC,
MSP ↔ FC), and ZMQ (everything internal to the autopilot side).

```
                  Gazebo Harmonic
                ┌───────────────────┐
                │ iris model +      │ UDP 9002 ← motor pwm
                │ BetaflightPlugin  │ UDP 9003 → IMU/pose/GPS/ESC
                └──────────┬────────┘
                           │
                  ┌────────▼────────┐
                  │  Betaflight SITL│ pre-built binary in bt_bringup/bin
                  │  (TCP 5761 MSP) │
                  └──┬──────────┬───┘
       websockify  │              │  TCP 5761 (raw MSP)
       ws://...:6761            (bt_app)
                  │              │
        Web Configurator   ┌─────▼─────┐
        (chrome app)       │  bt_app   │ py_trees + Context + controllers
                           │  Python   │ MSP client; RC at 50 Hz
                           └─┬──┬──┬───┘
              ZMQ (IPC)      │  │  │  ZMQ tcp:5555 (REP)
                             │  │  └───── bti CLI (Typer)
                             │  │
              ipc tracker    │  └── ipc sim sensors (lidar)
              result         │
                             └─── red_box_tracker / optical_flow_tracker
                                  (subscribe to /camera over gz transport,
                                   publish errors over ZMQ)
```

## Key Decisions

- **Real Betaflight, not a fake FC.** The autopilot does not implement
  flight stabilization; it commands the real Betaflight stack over MSP
  exactly as a human pilot's radio would (`bt_app/.../command_dispatcher.py`).
- **UDP for sim physics, TCP for MSP.** The Gazebo plugin uses fixed UDP
  ports 9002 (sim → SITL) and 9003 (SITL → sim) (`BetaflightPlugin.cc:329`,
  `:859`). MSP rides on TCP 5761 (UART1).
- **`websockify` exists solely to let the web Configurator connect**
  (`run_websockify.sh`) — Betaflight SITL only speaks raw TCP; the web
  Configurator only speaks WebSockets.
- **Two control paths share the dispatcher.** The behavior tree commands
  arming and discrete phase transitions via `TimedRcCommand`, while
  continuous controllers (takeoff, hover-yaw, visual) push RC into the
  same `MspCommandDispatcher` (`bt_app/main.py:71-94`).
- **ZMQ for in-process plumbing.** Tracker results, sensor scans, and
  parameter RPCs all use ZMQ (PUB/SUB and REP). No ROS.
- **Behavior tree as orchestrator.** `py_trees` Sequence chains the high-level
  flight phases (arm → takeoff → search → visual track → final track), but
  the actual closed loops live in dedicated controller threads
  (`bt_app/main.py:86-93`).

## Constraints

- Loop rate is 50 Hz (`bt_app/__init__.py:1` → `FREQ_HZ = 50.0`); all
  controllers tick on this period.
- Single host: every process binds to `127.0.0.1` (plugin sockets, MSP
  TCP, websockify, ZMQ tcp endpoints). Cross-host is not supported.
- Pre-built SITL binary in `bt_bringup/bin/betaflight_2025.12.2_SITL` — the
  workspace contains no build system for Betaflight itself.
- The autopilot assumes Betaflight is configured for `MSP_RX` (so
  `MSP_SET_RAW_RC` actually controls flight) and that AUX channel mappings
  match what the controllers send (arm=AUX1, angle mode=AUX2).

## Open Questions

- Failsafe path: if `bt_app` crashes mid-flight, Betaflight will eventually
  trip RX failsafe — but the maximum RC age before failsafe is not asserted
  anywhere in the repo. We must pin this when we rebuild.
- Time base: the plugin uses sim time, controllers use `time.monotonic()`.
  Real-time factor drift (which `gz_rtf_monitor.py` watches) silently
  corrupts PID `dt`. Not handled.
- The behavior tree owns the high-level FSM but `Context.set_state` is
  also called directly from `TakeoffController` / `HoverYawController` —
  the source of truth for "current flight phase" is ambiguous.
