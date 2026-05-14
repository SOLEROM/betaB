---
type: feature
title: "CRSF_BAUDRATE — CRSF UART Baud Rate"
status: developing
area: rc
since_version: "pre-4.0 (define); CRSF V3 negotiation since BF 4.3"
created: 2026-05-12
updated: 2026-05-12
tags: [feature, rc, crsf, elrs, serial, build-system, compile-time]
---

# CRSF_BAUDRATE

## What It Is

`CRSF_BAUDRATE` is a **compile-time `#define`** that sets the default
serial bit-rate Betaflight uses on the UART that talks to a CRSF receiver
(TBS Crossfire, [[ExpressLRS]], FrSky R9 with CRSF firmware, etc.).
CRSF is the serial framing used by those receivers to deliver RC channel
data, telemetry, and link statistics over a single full-duplex UART at
8N1.

Defined in `src/main/rx/crsf_protocol.h:33-37`:

```c
// Rev7 CRSF docs: UART runs at 400000 baud, 8N1 at 3.0 to 3.3V level.
// Rev10 CRSF docs: UART runs at 416666 baud, 8N1 at 3.0 to 3.3V level.
// Avoid using with ExpressLRS STM32 receivers.
#ifdef USE_CRSF_OFFICIAL_SPEC
#define CRSF_BAUDRATE           416666
#else
#define CRSF_BAUDRATE           420000
#endif
```

So BF ships with two possible values, both close to (but not exactly)
the spec:

| Build flag | Value | Why |
|---|---|---|
| `USE_CRSF_OFFICIAL_SPEC` defined | **416 666 baud** | TBS Rev10 spec (the "correct" number) |
| default — flag not defined | **420 000 baud** | The number ELRS firmwares use; ~0.8 % above spec but the receiver TX clocks at the same rate, so timing matches |

Why the mismatch with the spec? STM32 UART baud generators can't hit
416 666 exactly from common clock trees. 420 000 is what divides cleanly
on many MCUs, and the entire ELRS ecosystem standardised on it. Building
with `USE_CRSF_OFFICIAL_SPEC` is explicitly warned against for ELRS
STM32 receivers because both ends will drift apart at the edges of the
8-bit window.

## What It's For

This is the **single number that decides whether the FC can hear the
RX at all**. UART links are unforgiving: ±2-3 % skew between the two
ends and you get framing errors, dropped packets, or a totally silent
receiver. Once both sides agree on baud, CRSF frames flow:
`<sync 0xC8><len><type><payload><crc>` carrying RC channels at
~150-500 Hz depending on the link.

It's also the **lower bound** for [CRSF V3 baud
negotiation](#crsf-v3-runtime-negotiation): the receiver can ask the FC
to step *up* to a faster rate, but never below `CRSF_BAUDRATE`.

## Where It Is Stored

`CRSF_BAUDRATE` is **not** stored anywhere at runtime in the
traditional sense — it's a `#define`, so by the time the firmware is
linked the name is gone. See [[Finding Defines in Firmware Binaries]]
for the general principle.

Specifically, in the built `.elf` / `.hex` it shows up as:

- An **immediate operand** baked into instructions in `crsfRxInit()`
  (`rx/crsf.c:648`), `crsfRxUpdateBaudrate()` (`rx/crsf.c:682`),
  `getCrsfCachedBaudrate()` (`telemetry/crsf.c:146,150`), and
  `getCrsfDesiredSpeed()` (`telemetry/crsf.c:160`).
- Because the value (420 000 = `0x66900`) doesn't fit in a single
  Thumb-2 immediate, the compiler emits a `movw r0, #0x6900` + `movt r0, #0x6`
  pair, or a `ldr r0, [pc, #N]` referencing a literal pool entry near
  the function.

To find it in your build (from the `cross/` env):

```sh
make shell
arm-none-eabi-objdump -d obj/main/betaflight_STM32F405.elf \
    | grep -E -B1 '(movw|ldr).*0x6900|420000|0x66900' | head -40
arm-none-eabi-nm obj/main/betaflight_STM32F405.elf | grep -i crsfRx
```

The persistent counterpart (a *negotiated* baud, not the default) lives
elsewhere — see below.

## Where the *Negotiated* Baud Lives (CRSF V3)

When CRSF V3 negotiation is enabled, the agreed-on baud rate is
persisted in **STM32 backup-RAM** (battery-backed registers that
survive reset but not power loss without an FC backup cap):

- Slot: `PERSISTENT_OBJECT_SERIALRX_BAUD` (`drivers/persistent.h:40`).
- Writer: `crsfRxUpdateBaudrate()` at `rx/crsf.c:679` calls
  `persistentObjectWrite()`.
- Reader on boot: `getCrsfCachedBaudrate()` at `telemetry/crsf.c:141-151`
  reads the value and validates it against the `baudRates[]` table
  (`io/serial.c:225-228`), falling back to `CRSF_BAUDRATE` if the
  stored value is missing, corrupt, or *below* `CRSF_BAUDRATE`.

The valid step-up rates come from a fixed table:
```c
const uint32_t baudRates[BAUD_COUNT] = {
    0, 9600, 19200, 38400, 57600, 115200, 230400, 250000,
    400000, 460800, 500000, 921600, 1000000, 1500000, 2000000, 2470000
};
```

So a CRSF V3-negotiated link can run as fast as **2.47 Mbaud**, vs
the 420 kbaud default.

## How to Configure (CLI)

There is **no CLI variable** for `CRSF_BAUDRATE` itself — it's
compile-time only. The runtime knob is the *opt-in to negotiation*:

```
# Allow CRSF V3 to negotiate a higher baud (default OFF)
set crsf_use_negotiated_baud = ON
save
```

Backed by the PG entry at `cli/settings.c:874` →
`rxConfig_t.crsf_use_negotiated_baud` (`pg/rx.h:65`, default `false`
at `pg/rx.c:117`).

When ON, on boot `crsfRxInit()` picks up the cached negotiated baud
instead of `CRSF_BAUDRATE`:

```c
// src/main/rx/crsf.c:648-652
uint32_t crsfBaudrate = CRSF_BAUDRATE;
#if defined(USE_CRSF_V3)
crsfBaudrate = rxConfig->crsf_use_negotiated_baud
             ? getCrsfCachedBaudrate()
             : CRSF_BAUDRATE;
#endif
```

To clear a stuck negotiated baud: `set crsf_use_negotiated_baud = OFF`,
`save`, then power-cycle (so backup-RAM gets re-validated on next
init).

## How to *Change* the Default

Three escalating options:

### 1. Runtime: enable V3 negotiation (no rebuild)
The cleanest path if you have a CRSF V3-capable TX (modern ELRS ≥ 3.x,
TBS Crossfire firmware ≥ V3, RadioMaster on EdgeTX). Enable the CLI
flag above and the link will step up automatically.

### 2. Compile-time: switch to the official spec value
Build with `USE_CRSF_OFFICIAL_SPEC` defined — `CRSF_BAUDRATE` becomes
416 666. Add it via the build options, e.g.:

```sh
# in cross/
make build TARGET=STM32F405 EXTRA_FLAGS="-DUSE_CRSF_OFFICIAL_SPEC"
```

⚠️ The header comment explicitly warns against this with ELRS STM32
receivers — only do it if you've verified both ends are spec-compliant
hardware (TBS Crossfire RX, FrSky R9 on official firmware).

### 3. Compile-time: edit the define directly
Edit `src/main/rx/crsf_protocol.h:36` to a custom value and rebuild.
Almost never the right answer — see Gaps below for what breaks.

After any compile-time change, follow the pipeline in
[[ARM Cortex-M Firmware Build Process]]: rebuild, flash, and verify
the new value with the `objdump` recipe in the *Where It Is Stored*
section.

## How It Works

### Boot path (open the UART)

1. `crsfRxInit()` runs during RX subsystem init
   (`rx/crsf.c:633`).
2. `findSerialPortConfig(FUNCTION_RX_SERIAL)` returns the UART the
   user assigned to Serial RX in the Ports tab.
3. Baud is chosen — `CRSF_BAUDRATE` by default, or the persisted
   negotiated value if `crsf_use_negotiated_baud` is on and valid.
4. `openSerialPort(... CRSF_PORT_MODE, ... )` configures the UART
   at that baud, inverted if `serialrx_inverted = ON`.

### CRSF V3 negotiation

When a CRSF V3 TX sends a `CRSF_COMMAND_SUBCMD_GENERAL_CRSF_SPEED_PROPOSAL`
(`0x70`) frame, BF (`telemetry/crsf.c`):

1. Looks up the proposed baud in `baudRates[]`.
2. Replies with `CRSF_COMMAND_SUBCMD_GENERAL_CRSF_SPEED_RESPONSE` (`0x71`).
3. Calls `crsfRxUpdateBaudrate()` → `serialSetBaudRate()` to switch the
   UART live.
4. `persistentObjectWrite(PERSISTENT_OBJECT_SERIALRX_BAUD, baudrate)`
   saves it across reboots.
5. Recomputes `frameTimeNeededUs` proportionally (`rx/crsf.c:682`).
6. If telemetry-CRSF is built in and the new baud is *above*
   `CRSF_BAUDRATE`, swaps the telemetry task to event-driven mode for
   lower jitter (`rx/crsf.c:686-694`).

So `CRSF_BAUDRATE` keeps a structural role even when V3 is running:
it's the **safety floor** and the **frame-time reference**.

## Key Parameters

| Parameter | Default | Where | Notes |
|-----------|---------|-------|-------|
| `CRSF_BAUDRATE` (compile-time) | 420000 | `rx/crsf_protocol.h:36` | Or 416666 if `USE_CRSF_OFFICIAL_SPEC` is defined |
| `USE_CRSF_OFFICIAL_SPEC` (build flag) | undefined | build options | Switches the define to 416666 |
| `USE_CRSF_V3` (build flag) | enabled on most targets | `target/common_*.h` | Compiles in the negotiation machinery |
| `crsf_use_negotiated_baud` (CLI) | `OFF` | `rxConfig_t` | Runtime opt-in to use the negotiated baud on boot |
| `PERSISTENT_OBJECT_SERIALRX_BAUD` | — | STM32 backup-RAM | Where the negotiated baud is cached |

## Interactions

- Works with: [[ExpressLRS]] (the dominant consumer of the 420 000
  default), [[MSP Protocol]] (BF runs MSP over CRSF for
  configurator-over-radio).
- Depends on: [[ARM Cortex-M Firmware Build Process]] (the define is
  baked in at compile time), [[Finding Defines in Firmware Binaries]]
  (how to verify what value actually shipped in your `.hex`).
- Related runtime config (not defines): `serialrx_inverted`,
  `serialrx_provider`, the Ports-tab UART assignment.

## Gaps / Open Questions

> [!gap]
> What MCU clock trees can hit 416 666 cleanly? F4 and F7 derive UART
> from APB busses at 84 / 90 / 108 MHz — exact dividers would settle
> whether `USE_CRSF_OFFICIAL_SPEC` is safe on any STM32 part or only on
> select clock configurations.

> [!gap]
> Backup-RAM persistence depends on the FC having either VBAT or a
> backup cap. On boards without battery backup, does
> `PERSISTENT_OBJECT_SERIALRX_BAUD` survive across a normal
> battery-disconnect, or only across resets? Effect on V3 negotiation
> after power-cycling needs a test.

> [!gap]
> When `serialrx_inverted` is ON (UART inversion for legacy F3 / F4
> boards without hardware inversion), do the baud tolerances tighten?
> Software-driven inversion via DMA + timer is sensitive to clock jitter.

## Sources

- Source code: `src/main/rx/crsf_protocol.h`,
  `src/main/rx/crsf.c`, `src/main/telemetry/crsf.c`,
  `src/main/io/serial.c`, `src/main/pg/rx.{c,h}`,
  `src/main/drivers/persistent.h`, `src/main/cli/settings.c`
- TBS Crossfire protocol revisions (referenced in `crsf_protocol.h`
  header comment: Rev7 → 400 kbaud, Rev10 → 416 666 kbaud)
- Related wiki: [[ExpressLRS]], [[ARM Cortex-M Firmware Build Process]],
  [[Finding Defines in Firmware Binaries]]
