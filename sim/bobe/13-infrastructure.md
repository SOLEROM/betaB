---
noteId: "f55c5d904f8211f194a2c3b1eecd91b7"
tags: []

---

# Infrastructure

## Overview

The dev-host concerns: how the workspace bundles a reproducible build
environment with **Gazebo Harmonic**, **GPU passthrough** (NVIDIA + Ogre2
rendering), **X11 forwarding** for the visualizer, the Python toolchain
for `bt_app`, and editor integration (VS Code devcontainer + tasks).
Files: `docker-compose.yaml`, `Dockerfile`, `.devcontainer/devcontainer.json`,
`.devcontainer/Dockerfile.harmonic`, `.vscode/{settings.json,tasks.json}`.

## Key Decisions

- **Two Dockerfiles.** Root `Dockerfile` targets `gazebo:jetty-full`;
  devcontainer uses `gazebo:harmonic-full`. The compose file points at
  `.devcontainer/Dockerfile.harmonic` — Harmonic is the live one.
  Jetty file appears to be an older / parallel experiment.
- **GPU passthrough via NVIDIA Container Toolkit.** `docker-compose.yaml`
  reserves `capabilities: [gpu]`, sets `NVIDIA_VISIBLE_DEVICES=all`,
  `NVIDIA_DRIVER_CAPABILITIES=all`, and bind-mounts
  `/dev/nvidia0`, `/dev/nvidiactl`, `/dev/nvidia-modeset`, `/dev/dri`.
  The Dockerfile pulls `libglvnd0`, `libgl1`, `libglx0`, `libegl1` to
  match.
- **X11 forwarding via `/tmp/.X11-unix` mount + `DISPLAY` env.**
  `QT_X11_NO_MITSHM=1` set to avoid MIT-SHM crashes (`Dockerfile:20`).
- **`network_mode: host`** for the gazebo service — all ports (5761,
  6761, 9002, 9003, 5555, ipc sockets) share the host's namespace.
  Simplifies inter-process wiring at the cost of port isolation.
- **`hostname: gz` + `GZ_DISCOVERY_SERVER=gazebo` + partition.** Pins
  the gz-transport discovery to a known peer so multiple compose stacks
  don't cross-talk.
- **Workspace bind-mount** `.:/workspace:cached`. Edits on the host are
  immediately visible inside the container. `./.gz` is also mounted to
  persist gz-sim resource cache.
- **Non-root user `user` (UID 1000)** with passwordless sudo
  (`Dockerfile.harmonic:22-39`). Deletes the default `ubuntu` user first
  to avoid UID collisions on newer Ubuntu base images.
- **Python venv at `/workspace/.venv`** with `bt_app` installed editable
  (`Dockerfile.harmonic:70-79`). All Python work uses that venv.
- **VS Code tasks define the launch surface.** `gazebo`, `sitl`,
  `websockify`, `Start SITL` (composite). No tasks for `bt_app` or
  the trackers — those are run manually.

## Constraints

- Linux + NVIDIA only. The compose file's GPU stanza assumes
  `nvidia-container-runtime`; on AMD or CPU-only hosts it must be
  stripped.
- `network_mode: host` forecloses parallel stacks on the same host
  (port collisions). Tests can't easily run two instances side by side.
- Anyone in the `docker` group can run this container with full GPU /
  X11 access — local-dev posture only, not multi-tenant.
- `.gz` cache is host-writable; permission mismatches across hosts
  can corrupt resources.

## Open Questions

- Drop the `jetty` Dockerfile or keep as a fallback? It is identical
  except for the base image and missing gz-dev packages.
- Why `--build-arg USERNAME=user` instead of matching the invoking host
  UID/GID? Files written by the container appear owned by `1000:1000`
  which may not be the host user.
- VS Code tasks are the only launch surface — no `Makefile` or
  `justfile`. We'll likely want a script wrapper for headless / CI use.
- No CI pipeline is defined anywhere; tests are local-only.
