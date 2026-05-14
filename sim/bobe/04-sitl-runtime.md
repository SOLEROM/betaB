---
noteId: "7da3ddf04f8211f194a2c3b1eecd91b7"
tags: []

---

# SITL Runtime & Launch

## Overview

Runtime concerns: how the Betaflight SITL binary is **acquired**, how the
companion processes (websockify, gz sim, bt_app) are **started**, and the
**dev environment** that hosts them. Rob's workspace ships a pre-built
Betaflight ELF, a docker-compose-based devcontainer for Gazebo, three
shell launch scripts under `bt_bringup/launch/`, and VS Code tasks that
group them. There is no Makefile and no Betaflight build pipeline.

## Key Decisions

- **Pre-built binary, version-pinned.** Two ELFs are checked into git:
  `betaflight_2025.12.2_SITL` and `betaflight_2025.12.2_SITL.24`
  (`bt_bringup/bin/`). The 2025.12 tag implies upstream Betaflight master,
  not 4.5-maintenance. Decoupling SITL builds from the workspace is
  deliberate — there's no cross-compile chain in the dev image.
- **VS Code as the entry point.** `.vscode/tasks.json` exposes
  `Start SITL` (depends-on: `websockify`, `sitl`) and a separate `gazebo`
  task. There is **no** single command that brings up the whole stack —
  user is expected to start gazebo, then "Start SITL", then run bt_app
  manually.
- **`websockify` is a Python venv install via `bt_app[dev]`.** The script
  sources `venv/bin/activate` (`run_websockify.sh`), so the dev tooling
  is wedged into the bt_app project. It maps `ws://127.0.0.1:6761 →
  tcp://127.0.0.1:5761`.
- **devcontainer = docker-compose service.**
  `.devcontainer/devcontainer.json` points at `docker-compose.yaml`,
  which builds from `.devcontainer/Dockerfile.harmonic` (a Gazebo-Harmonic
  full image). All work happens inside the container at `/workspace`.
- **The autopilot is installed editable.** The Dockerfile runs
  `pip install -e .` and `pip install -e .[dev]` in
  `/workspace/.venv` (`Dockerfile.harmonic:73-79`) so any Python edit is
  immediately live.
- **Run scripts are minimal.**
  - `run_sitl.sh` just execs the pinned binary.
  - `run_gz.sh` exports `GZ_SIM_SYSTEM_PLUGIN_PATH=/workspace/bt_gazebo/bin`
    and `GZ_SIM_RESOURCE_PATH=/workspace/bt_gazebo/models:/workspace/bt_gazebo/worlds`,
    then `gz sim -v 4 -r betaloop_iris_betaflight_demo_harmonic.sdf`.
  - `run_websockify.sh` activates the venv and binds the WebSocket→TCP
    bridge.

## Constraints

- Binary is **Ubuntu-24-noble-glibc-linked**; running it on a different
  base image (or bare host) requires matching glibc.
- All scripts hardcode `/workspace/...` paths — must run inside the
  devcontainer (or with an identical bind mount).
- `eeprom.bin` is written to the cwd of the SITL process; checked-in
  `eeprom.bin` at repo root means the first `cd` matters for whether
  state is fresh or pre-configured.
- Web Configurator requires `ws://127.0.0.1:6761` — IP/port hardcoded in
  `run_websockify.sh`.

## Open Questions

- The two SITL binaries differ only by a `.24` suffix — no documentation
  on the difference; one is probably a debug or alternate-target build.
- No "stop" path. Killing the launch task leaves SITL running and holding
  TCP 5761. We need a deliberate shutdown hook in our rebuild.
- No build instructions for the SITL binary itself — porting to BF 4.5 or
  rebuilding from source is unspecified.
- ws://6761 is not authenticated; anyone on the host can drive the FC.
