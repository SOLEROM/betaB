# cross/ — Betaflight Docker Build Environment

Self-contained, reproducible build environment for compiling [Betaflight](https://github.com/betaflight/betaflight) firmware. Everything the upstream `Makefile` expects (ARM GNU Toolchain 13.3.Rel1, GNU make, python3, ruby, etc.) lives in a Docker image — the host only needs `docker` and `make`.

Verified: produces `betaflight_<version>_STM32F405.hex` (~1.7 MB) on the host.

## Files

| File         | Purpose                                                        |
| ------------ | -------------------------------------------------------------- |
| `Dockerfile` | Ubuntu 22.04 + arm-none-eabi-gcc 13.3.1 + build deps           |
| `Makefile`   | Driver that builds the image and runs `make` / shell inside it |
| `README.md`  | This file                                                      |

## Prerequisites

- Docker (with BuildKit; default in modern installs)
- GNU make on the host
- Betaflight source. Default `PROJECT` is `../betaflight` (the submodule at the repo root). If empty, hydrate it from the repo root:

  ```sh
  cd /mnt/betab
  git submodule update --init --recursive
  ```

  Recommended: run the host-side hydrate once. It pulls submodules with your normal git/SSH credentials and the `cross/` env stays focused purely on compiling. The `make hydrate` target is still available as a fallback (works on a fresh clone with only docker installed).

## Quickstart

```sh
cd /mnt/betab/cross

# 1) Build the toolchain image (one-time, ~5–10 min, ~1.5 GB)
make image

# 2) Compile a target — output lands in ../betaflight/obj/
make build TARGET=STM32F405

# 3) Need to poke around the env? Open a shell
make shell
```

Built firmware artifacts (e.g. `betaflight_<version>_STM32F405.hex`) appear in `$(PROJECT)/obj/` on the host because the project folder is mounted as a volume.

## Make targets

| Target    | What it does                                                                   |
| --------- | ------------------------------------------------------------------------------ |
| `image`   | Builds the `betab-cross:latest` docker image                                   |
| `hydrate` | Runs `git submodule update --init --recursive` (serial) inside the container   |
| `build`   | Hydrates submodules, then runs `make -j TARGET=$(TARGET)` inside the container |
| `shell`   | Drops you into an interactive bash in the env, cwd = container workdir         |
| `clean`   | Runs `make clean TARGET=$(TARGET)` inside the container                        |
| `version` | Prints `arm-none-eabi-gcc --version` (sanity check)                            |
| `check`   | Verifies the image exists and `PROJECT` looks like betaflight                  |
| `help`    | Prints usage with current variable values                                      |

> **Why hydrate is a step on its own:** Betaflight's Makefile hydrates each required submodule as an individual `.git` stamp file. Under `make -j`, those rules run in parallel and race on the gitdir's `config.lock`, breaking the build. We run `git submodule update --init --recursive` once, serially, before the parallel build. If you've already hydrated on the host, the step is a near-instant no-op.

## Variables

| Var       | Default              | Meaning                                  |
| --------- | -------------------- | ---------------------------------------- |
| `PROJECT` | `../betaflight`      | Host path mounted into the container     |
| `TARGET`  | `STM32F405`          | Betaflight build target (MCU / FC board) |
| `IMAGE`   | `betab-cross:latest` | Docker image tag                         |
| `JOBS`    | `$(nproc)`           | Parallel `make -j` jobs                  |

Override on the command line:

```sh
make build PROJECT=/abs/path/to/betaflight TARGET=STM32H743 JOBS=8
```

## Building from an interactive shell

```sh
make shell
# you land in /super/betaflight (or /work for non-submodule projects)

make targets                    # list all supported FC targets
make configs                    # list available unified configs
make -j$(nproc) TARGET=STM32F405

# build from a unified config instead:
make -j$(nproc) CONFIG=<config-name>

# clean & rebuild:
make clean TARGET=STM32F405

arm-none-eabi-gcc --version     # sanity check
```

Artifacts written under `obj/` persist to the host via the volume mount.

## How it works

- The Dockerfile downloads the official ARM GNU Toolchain (`13.3.Rel1`) for the container's architecture (amd64 or arm64) and installs it under `/opt/arm-toolchain`. This is the **same version pinned by `betaflight/mk/tools.mk`** (`GCC_REQUIRED_VERSION := 13.3.1`), so the in-container build skips toolchain installation entirely.
- A non-root user `dev` is created matching the host's UID/GID. Files written into the mounted volume stay owned by the invoking host user.
- `make build` essentially does:
  ```
  docker run --rm -v <PROJECT-parent>:/super -w /super/<name> betab-cross \
      git submodule update --init --recursive
  docker run --rm -v <PROJECT-parent>:/super -w /super/<name> betab-cross \
      make -j$(nproc) TARGET=<TARGET>
  ```

## Submodule layout note

If `PROJECT` is a git submodule (the default `../betaflight` is), its `.git` is a *file* pointing to a gitdir in the parent superproject (`../.git/modules/<name>`), and that gitdir's own config holds a relative path back to the worktree. Mounting only the project folder breaks both indirections.

The Makefile detects this and **mounts the parent of `PROJECT` at `/super`** inside the container, mirroring the host layout 1:1. The container workdir becomes `/super/<name>` and every relative path (the `.git` pointer, the gitdir's `core.worktree`, nested submodules) resolves naturally. Side effect: sibling folders of `PROJECT` are visible inside the container under `/super/`.

If you point `PROJECT` at a directory with a real `.git/` directory (not a submodule), the Makefile mounts `PROJECT` directly at `/work` — no parent mount.

## Cleaning up

```sh
make clean TARGET=STM32F405          # clean betaflight build artifacts for a target
docker image rm betab-cross:latest    # remove the toolchain image
```

## Troubleshooting

- **`PROJECT does not look like a betaflight checkout`** — pass `PROJECT=/full/path/to/betaflight` or run `git submodule update --init --recursive` at the repo root.
- **`fatal: not a git repository: /super/.git/modules/...`** or **`cannot chdir to ...`** — the parent-directory mount didn't compute. Make sure you're invoking the Makefile in this folder so the submodule detection runs.
- **`error: could not lock config file ... config: File exists`** during build — submodule hydration race. Run `make hydrate` first (or hydrate from the host once), then `make build`.
- **`make: arm-none-eabi-gcc: command not found`** in container — image is stale; rebuild with `make image`.
- **Permission errors on `obj/`** — image was built with mismatched UID/GID; rebuild with `make image` from the same user.
- **Slow first build** — the ARM toolchain tarball is ~150 MB. Subsequent `make build` runs reuse the image.

## What's verified

| Step                                     | Result                                                                       |
| ---------------------------------------- | ---------------------------------------------------------------------------- |
| `make image`                             | Builds successfully on amd64 and arm64 (~1.9 GB image)                       |
| `make hydrate`                           | Pulls `src/config`, `dronecan/libcanard`, nested `avr-can-lib`               |
| `make build TARGET=STM32F405`            | Produces `../betaflight/obj/betaflight_<version>_STM32F405.hex` (~1.7 MB)    |
| `make shell` → `arm-none-eabi-gcc -v`    | Reports GCC 13.3.1 (matches `betaflight/mk/tools.mk`)                        |
