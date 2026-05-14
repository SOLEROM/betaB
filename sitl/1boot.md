---
noteId: "7922e9b04f8211f194a2c3b1eecd91b7"
tags: []

---

# First-boot lessons

Notes from the first end-to-end run of our Dockerised SITL against
Betaflight 4.5-maintenance + desktop Configurator 10.10.0.

---

## 1. The port off-by-one

SITL's TCP-UART bridge (`src/main/drivers/serial_tcp.c`) binds at:

```c
dyad_listenEx(s->serv, NULL, BASE_PORT + id + 1, 10);   // BASE_PORT = 5760
```

`id` is the **zero-indexed** UART number, so:

| UART | Host port |
|------|-----------|
| UART1 (MSP) | **5761** |
| UART2 | 5762 |
| UART3 | 5763 |
| UART4 | 5764 |

The upstream `target/SITL/README.md` says "`tcp://127.0.0.1:576x` when
port been open" — accurate but misleading; `x=0` is **not** a real UART.
Configurator must point at `tcp://127.0.0.1:5761`, not `:5760`.

If `make run` ever prints `bind port 5761 for UART1`, the listener is
live. If you don't see that line, MSP isn't bound and Configurator
will hit `Chrome API Error: net::ERR_FAILED`.

---

## 2. Tool-version check on 4.5

`mk/tools.mk` on 4.5-maintenance does a **strict equality** version
check on `arm-none-eabi-gcc` before any include can override it — even
for SITL, which doesn't use the ARM toolchain.

Bypass: set `ARM_SDK_DIR` to any existing directory. The check is just
`[ -d "$(ARM_SDK_DIR)" ]`; if it passes, the version check is skipped.
`mk/mcu/SITL.mk` then clears `ARM_SDK_PREFIX`, and `CROSS_CC` resolves
to plain `gcc` (host).

Our `make build` already passes `ARM_SDK_DIR=/tmp`. Harmless on master
(master uses `PLATFORM_SDK := none` and never reaches that branch).

---

## 3. Configurator vs firmware version skew

| Tree | FW version reported | Configurator 10.10.0 |
|------|---------------------|----------------------|
| `betaflight/` (master) | 26.6.0 | Warning: "does not support firmware 26.6.0" |
| `betaflight_4.5/` (4.5-maintenance) | 4.5.x | Clean |

Stick with the 4.5 tree for GUI work with the 10.10.0 Configurator.
Master is fine for CLI-only research, but the GUI will mis-render new
fields and refuse some operations.

---

## 4. Motor protocol must be set

On a fresh SITL (no `eeprom.bin`), the first connect raises:

> there is no motor output protocol selected.

SITL has no real ESCs, so the protocol choice is symbolic — but the
config validator still demands one. **PWM** is the canonical SITL
pick (no timing assumptions, no DShot bit-banging).

### Set via CLI (fastest)

In Configurator → **CLI** tab:

```
set motor_pwm_protocol = PWM
save
```

`save` writes `eeprom.bin` and reboots SITL. The reboot in SITL is a
plain `exit()` — the container's `--rm -it` shell stays attached, but
the SITL process is gone. You need to `Ctrl-C` and `make run` again.

### Set via GUI

Left sidebar → **Configuration** tab → *ESC/Motor Features* →
**Motor PWM protocol** = `PWM` → *Save and Reboot*.

> Caveat we hit: saving from the GUI on the very first connect can
> write default values into newly-introduced fields that the running
> firmware then refuses on the next boot, causing the **"No
> configuration received within 10 seconds"** symptom. CLI-only first
> save is the safer path.

---

## 5. `eeprom.bin` — where it lives & when to nuke it

- Path: `$(BF_SRC)/eeprom.bin` (mounted into the container at
  `/src/eeprom.bin`).
- Size: 8192 bytes (set by `__FLASH_CONFIG_Size` in `SITL.ld`).
- Lifetime: persists across `make run` invocations.

**Reset to factory:**
```bash
rm /mnt/betab/betaflight_4.5/eeprom.bin
make -C /mnt/betab/sitl run BF_SRC=/mnt/betab/betaflight_4.5
```

**When to nuke it:**
- `Configurator: No configuration received within 10 seconds` after
  `make run` (saved config is wedging boot).
- Switching `BF_SRC` between different BF major versions — EEPROM
  layout isn't compatible across major bumps.
- After upgrading the firmware in-place — same reason.

---

## 6. End-to-end flow that works

```bash
# Terminal A: build + run SITL
cd /mnt/betab/sitl
make image                                          # one-time
make build BF_SRC=/mnt/betab/betaflight_4.5
make run   BF_SRC=/mnt/betab/betaflight_4.5
# wait for: "bind port 5761 for UART1"

# Terminal B: sanity-check the port (optional)
nc -z 127.0.0.1 5761 && echo "5761 open"

# Configurator: Manual Selection, Port = tcp://127.0.0.1:5761, Connect
# Warning about no motor protocol → CLI tab:
#   set motor_pwm_protocol = PWM
#   save
# SITL exits. Ctrl-C in Terminal A, `make run` again.
# Reconnect Configurator — warning gone. SITL is now usable.
```

---

## 7. Stopping SITL — three ways

1. **Ctrl-C** in the `make run` terminal (preferred — `make run` is foreground).
2. **`make stop`** from another shell — uses the named container.
3. **`docker stop betaflight-sitl`** directly.
