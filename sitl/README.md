---
noteId: "14f360004f7811f194a2c3b1eecd91b7"
tags: []

---

# Betaflight SITL — Dockerised builder & runner

A self-contained Docker setup that compiles the Betaflight
**SITL** (Software-In-The-Loop) target from a Betaflight source tree
and runs the resulting native binary with the right ports exposed.

The container only holds the **toolchain**. Your source tree is mounted
in, build artifacts land back on the host, and `ccache` is persisted in
a named volume — so incremental rebuilds are fast.

---

## What gets built

Betaflight's SITL target is a normal Linux ELF (not ARM firmware) that
emulates the flight controller. It speaks:

- **TCP `576x`** — one port per UART. SITL's formula is `5760 + uart_id + 1`,
  so **UART1 → 5761**, UART2 → 5762, UART3 → 5763, UART4 → 5764. The
  Configurator / a TCP-MSP client talks to UART1 (MSP) on **5761**.
- **UDP `9002`** — sim → SITL: IMU / sensor packets.
- **UDP `9003`** — SITL → sim: motor PWM outputs.

It persists config in an `eeprom.bin` file in the working directory.

Output binary path on the host:
```
<BF_SRC>/obj/main/betaflight_SITL.elf
```

---

## Prerequisites

- Docker (Engine 20.10+; rootless or rootful both fine).
- A Betaflight source checkout that contains `src/platform/SIMULATOR/`.
  In this repo that's the sibling `betaflight/` submodule:
  ```bash
  git submodule update --init --recursive betaflight
  ```

---

## Quick start

From `/mnt/betab/sitl/`:

```bash
# 1. Build the toolchain image (one-time, ~5 min)
make image

# 2. Compile SITL  (uses ../betaflight by default)
make build

# 3. Run it with UART + sim ports exposed
make run
```

After `make run`, point the Betaflight Configurator at
**`tcp://127.0.0.1:5761`** to connect.

To **stop** SITL:

- Ctrl-C in the `make run` terminal (preferred — `make run` is foreground), or
- `make stop` from another shell, or
- `docker stop betaflight-sitl` directly.

---

## Pointing at a different source tree

By default the Makefile assumes the source is at `../betaflight`
(relative to this directory). Override per-invocation:

```bash
make build BF_SRC=/path/to/my/betaflight-fork
make run   BF_SRC=/path/to/my/betaflight-fork
```

The path must contain Betaflight's top-level `Makefile` and the
`src/platform/SIMULATOR/` tree (`make verify-src` runs that check).

---

## Makefile rules

| Rule | What it does |
|------|--------------|
| `make help` | Print rules + current config. |
| `make image` | Build the toolchain Docker image. |
| `make build` | Run `make TARGET=SITL` inside the container. Output goes to `$(BF_SRC)/obj/main/betaflight_SITL.elf`. |
| `make run` | Launch the built SITL with TCP `5761–5764` + UDP `9002/9003` reachable on the host. Foreground; Ctrl-C to stop. Defaults to `--network host` for low latency (override with `NET_MODE=bridge`). |
| `make stop` | Stop the running SITL container by name (use from another shell if `make run` is detached or wedged). |
| `make shell` | Interactive bash in the builder image, source mounted at `/src`. |
| `make clean` | `make clean` inside the container — drops `obj/`. |
| `make nuke` | `clean` + remove the Docker image and the ccache volume. |
| `make ccache-stats` | Show ccache hit/miss/size. |

### Useful overrides

```bash
# Build with extra OPTIONS (Betaflight build flags)
make build OPTIONS="SITL_STATIC"

# Limit parallel jobs (defaults to `nproc`)
make build JOBS=2

# Use a different target name (only useful if upstream adds variants)
make build TARGET=SITL_X_PLANE

# Run with explicit Docker port-forwarding instead of host networking
# (use this on macOS / Docker Desktop, where --network host has different semantics)
make run NET_MODE=bridge
```

---

## Using the running SITL

### Connect with the Configurator

1. Launch SITL: `make run`
2. Open Betaflight Configurator → manual connection → choose
   **TCP** and enter `tcp://127.0.0.1:5761` (UART1).
3. Connect. From here it behaves like a real FC — CLI, tabs, save,
   profile dump etc. all work.

### Connect with `nc` / a custom MSP client

```bash
nc 127.0.0.1 5761            # raw MSP over TCP on UART1
```

### Driving from a simulator (Gazebo etc.)

The sim must:

- Receive motor outputs on **`udp://<sitl-host>:9003`**.
- Send IMU/sensor packets to **`udp://<sitl-host>:9002`**.

For the upstream Gazebo `ArduCopterPlugin` recipe see
`betaflight/src/platform/SIMULATOR/target/SITL/README.md`.

### Where does `eeprom.bin` live?

In the container, the working directory is `/src`, so the EEPROM file
is written to `$(BF_SRC)/eeprom.bin`. It's `8192` bytes by default
(adjust `__FLASH_CONFIG_Size` in
`src/platform/SIMULATOR/link/sitl.ld` if you need more).

Delete the file to factory-reset SITL.

---

## How it works internally

- **Toolchain image** — Ubuntu 24.04 + `build-essential`, `make`,
  `python3`, `ruby`, `git`, `ccache`. No ARM toolchain — SITL compiles
  with the host gcc (`PLATFORM_SDK := none` in `SIMULATOR/mk/SITL.mk`).
- **UID/GID alignment** — `make image` passes the host UID/GID as build
  args so files written into the mounted source tree are owned by you,
  not root.
- **ccache** — mounted from the named volume `bf-sitl-ccache` at
  `/ccache`. First build is ~3-5 min; later builds (one-file edits) are
  seconds.
- **Networking** — `make run` defaults to `--network host` (Linux). The
  container shares the host's network namespace, so SITL's bind to
  `0.0.0.0:5761` *is* the host's `127.0.0.1:5761` — no `docker-proxy`
  in the path. This noticeably improves Configurator responsiveness
  (MSP polling lag → near-zero). Set `NET_MODE=bridge` to fall back to
  explicit `-p` port forwarding (needed on macOS Docker Desktop or
  Windows). In bridge mode the host port range can be overridden via
  `UART_PORTS`; in host mode the container can only bind whatever ports
  are free on the host directly.
- **No image rebuild needed for source changes** — the source is a
  bind mount, not a `COPY` layer. Only `make image` rebuilds the
  toolchain layer.

---

## Configurator responsiveness

If the desktop Configurator feels sluggish while connected to SITL,
the bottleneck is almost never SITL itself — `docker stats` typically
shows SITL at single-digit % CPU because its main loop calls
`microsleep(1000)` per tick. The lag comes from two places:

1. **The Docker networking path.** Bridge mode routes every MSP packet
   through `docker-proxy` in userland. Host mode (the default in our
   `make run`) bypasses it. Confirm with:
   ```bash
   docker inspect betaflight-sitl --format '{{.HostConfig.NetworkMode}}'
   # expect: host
   ```
   If you're on `bridge` and want host:
   ```bash
   make stop
   make run NET_MODE=host
   ```

2. **Configurator polling-heavy tabs.** *Setup*, *Sensors*, *Receiver*,
   and *Motors* poll MSP at ~50 Hz. The *CLI*, *Ports*, *Configuration*,
   and *Modes* tabs don't continuously poll — they feel instant. For
   research work, stay on CLI; switch to a polling tab only when you
   need live telemetry.

Other things worth trying:

- **Close the log panel** at the bottom of the Configurator window —
  it re-layouts on every MSP frame.
- **Upgrade the Configurator.** 10.10.0 runs on a 2020-era Electron
  (Chromium ~87). The 10.11+ releases are noticeably snappier and stay
  compatible with BF 4.5 firmware.

---

## Troubleshooting

**`Error: <BF_SRC>/Makefile not found`** — your `BF_SRC` doesn't point at
a Betaflight checkout. Pass the right path or `git submodule update --init`.

**`Error: <BF_SRC>/src/platform/SIMULATOR missing`** — the submodule
fetched but the platform sources weren't pulled. Run
`git submodule update --init --recursive` inside the Betaflight repo.

**Build fails with linker errors about `-lrt` on macOS** — this is a
known upstream wrinkle; the SIMULATOR mk already handles it for native
mac builds, but the *Docker* path is Linux, so this shouldn't hit you.

**Port already in use** — something on the host is already bound. Override:
```bash
make run UART_PORTS="6761 6762 6763 6764"
```
The container side stays at `576x`; only the host-visible port changes.

**Configurator: `Chrome API Error: net::ERR_FAILED`** — SITL isn't
listening on the port you targeted. Verify:
```bash
nc -z 127.0.0.1 5761 && echo "5761 open"   # MSP / UART1
```
If closed, check the `make run` stdout for `bind port 5761 for UART1`.
Remember UART1 is on **5761**, not 5760 (off-by-one in the SITL port formula).

**`arm-none-eabi-gcc not in the PATH` on a 4.5-era tree** — older
trees check for the ARM toolchain unconditionally even when building
SITL. The Makefile sidesteps this by passing `ARM_SDK_DIR=/tmp` to
the inner build (any existing dir works — the SITL MCU mk clears the
prefix afterwards). Already wired into `make build`; if you call
`make` inside the container by hand, add `ARM_SDK_DIR=/tmp` yourself.

**Configurator won't connect** — confirm SITL is actually printing
`bind port 5761 for UART1` on stdout, then try `nc 127.0.0.1 5761`. If
that works but Configurator doesn't, it's the Configurator's TCP-mode
setting, not SITL.

---

## Files in this directory

| File | Purpose |
|------|---------|
| `Dockerfile` | Toolchain image (Ubuntu 24.04 + native gcc + ccache). |
| `Makefile` | Driver — `image / build / run / shell / clean / nuke`. |
| `.dockerignore` | Only the `Dockerfile` is in the build context (source is mounted at run-time). |
| `README.md` | This file. |
