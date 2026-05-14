---
noteId: "397e7e504f8211f194a2c3b1eecd91b7"
tags: []

---

ref git : https://github.com/robobe/bt_ws

# bobe — design tree (reverse-engineered from `/tmp/rob/bt_ws`)

A reverse-engineered design tree of "rob's" Betaflight + Gazebo workspace.
The system is a **visual-tracking quadrotor demo**: Betaflight SITL flies a
simulated iris quadrotor inside Gazebo Harmonic, controlled by a Python
autopilot built on a behavior tree. The autopilot reads simulated sensors
(camera, lidar) over ZMQ, runs a vision tracker on a red box, closes
altitude/yaw/visual loops in Python, and steers Betaflight by streaming
synthetic RC over MSP. A live parameter service (YAML + ZMQ REP) and a
Typer CLI tune the controllers at runtime.

This tree is **descriptive of the existing repo**, not a forward plan.
Decisions inferred from code carry citations like `bt_app/.../foo.py:NN`.
"Open Questions" sections list assumptions the source did not make
explicit — these are the places where, when we rebuild, we will have to
choose deliberately.

## Files

- `01-architecture.md` — system overview, process topology, message flow.
- `02-simulation-world.md` — Gazebo world, vehicle model, motor mapping.
- `03-sim-bridge-plugin.md` — `BetaflightPlugin.cc` UDP bridge between Gazebo and SITL.
- `04-sitl-runtime.md` — SITL binary, websockify, devcontainer, launch scripts.
- `05-msp-stack.md` — MSP v1/v2 codec, transports, client, scheduled-command dispatcher.
- `06-flight-state-machine.md` — `py_trees` behavior tree, `Context`, `State` enum.
- `07-controllers.md` — PID, takeoff, hover-yaw, visual controller, RC mapper.
- `08-sensors-bridges.md` — Gazebo→ZMQ bridges (lidar, camera) and sim sensor consumer.
- `09-vision-trackers.md` — Red-box and optical-flow trackers publishing on ZMQ.
- `10-parameter-service.md` — YAML-backed parameter store, ZMQ REP server, change events.
- `11-cli.md` — `bti` Typer CLI talking to the parameter server.
- `12-gui-scaffold.md` — PyQt6 MVP scaffold (currently a placeholder).
- `13-infrastructure.md` — docker-compose, NVIDIA/X11 passthrough, VS Code tasks.
