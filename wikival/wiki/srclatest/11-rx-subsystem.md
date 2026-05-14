---
type: architecture
status: stable
created: 2026-05-14
updated: 2026-05-14
tags: [rx, receiver, sbus, crsf, expresslrs, srxl, spi-rx]
source-commit: 6434dd725
---

# 11 — RX Subsystem

`src/main/rx/` is the largest single feature directory in the firmware — around 70 files implementing every receiver protocol Betaflight has ever supported. It splits cleanly into two halves:

- **Serial RX** — protocols that arrive as a byte stream over UART. The RX module is wired to a peripheral; the radio (SBUS, CRSF, GHST, iBus, FPort, SRXL2, …) does its own RF and just exports digital channel data.
- **SPI RX** — protocols where Betaflight talks directly to the RF chip itself and implements the full radio stack in firmware. The FC *is* the receiver. Used with bare RF modules: CC2500, CYRF6936, A7105, NRF24L01, SX127x, SX1280.

Plus a dispatcher (`rx.c`) that sits in front of both and exposes `rcData[]` to the rest of the firmware.

## The dispatcher — `rx.c`

`src/main/rx/rx.c` (~1100 lines):

- Owns `rcData[MAX_SUPPORTED_RC_CHANNEL_COUNT]` (the post-mapped channel array consumed by `fc/rc.c`).
- Polls the active provider once per scheduled tick via `rxRuntimeState->rcFrameStatusFn()`.
- Applies channel-map (`rxConfig()->rcmap`) — the user can reorder channels.
- Applies failsafe values when frames stop coming.
- Tracks RSSI source: from a dedicated channel, from frame-error count, from the protocol itself (CRSF/GHST embed it), or from ADC.
- Exposes `rxIsReceivingSignal()`, `rxFrameTimedOut()`, `rcChannelLetters()`.

The frame-checker is a `checkFunc` so `TASK_RX` doesn't wake up the dispatcher loop when no frame is pending:

```c
static bool rxUpdateCheck(timeUs_t currentTimeUs, timeDelta_t currentDeltaTimeUs) {
    return rxRuntimeState.rcFrameStatusFn() & RX_FRAME_COMPLETE;
}
```

The polling rate of `TASK_RX` is dynamic — when frames are flowing, the scheduler runs the task as often as needed; when they stop, the check function fails and the task is deprioritised. The "RX expedite" feature scales down the task's estimated cost when it's been waiting, so it scheduled aggressively as soon as a frame arrives.

## Serial RX protocols

Each protocol has a `<name>.c/.h` pair. The pattern is identical:

```c
bool <protocol>Init(const rxConfig_t *cfg, rxRuntimeState_t *state);
// hooks .rcFrameStatusFn, .rcReadRawFn, .rcGetFrameTimeUsFn on state
// opens a serial port via openSerialPort(FUNCTION_RX_SERIAL, ...)
```

Bytes arriving on the serial port flow into a per-protocol state machine that decodes frames into channel values; the state machine sets `RX_FRAME_COMPLETE` flag and stashes the channel data; the dispatcher picks it up.

### Catalogue

| File pair | Protocol | Notes |
|-----------|----------|-------|
| `sbus.c`, `sbus.h`, `sbus_channels.c`, `sbus_channels.h` | Futaba S.BUS | 100 kbit, inverted, 16 channels @ 11 bits. Most legacy of the modern protocols. SBUS2 variant supported. |
| `crsf.c`, `crsf.h`, `crsf_protocol.h` | CRSF (Crossfire / ExpressLRS) | 416–921 kbaud, bidirectional, supports MSP tunnel + telemetry. Modern default. |
| `ghst.c`, `ghst.h`, `ghst_protocol.h` | Ghost (GHST) | ImmersionRC's ghost link, similar to CRSF in spirit. |
| `ibus.c`, `ibus.h` | Flysky iBus | 14 channels, RX+telemetry sharing a UART via inverter. |
| `fport.c`, `fport.h` | FrSky FPort | Combined RX + telemetry on one UART, bidirectional. |
| `spektrum.c`, `spektrum.h` | Spektrum DSM2 / DSMX | Serial mode of a Spektrum receiver. |
| `srxl2.c`, `srxl2.h`, `srxl2_types.h` | Spektrum SRXL2 | Newer Spektrum digital protocol. |
| `sumd.c`, `sumd.h` | Graupner SUMD | HoTT serial format. |
| `sumh.c`, `sumh.h` | Graupner SUMH | Older HoTT format. |
| `xbus.c`, `xbus.h` | JR XBus | XBus Mode B. |
| `jetiexbus.c`, `jetiexbus.h` | Jeti ExBus | Jeti DC/DS transmitters. |
| `mavlink.c`, `mavlink.h` | MAVLink RC channels | For autopilot/ground-station-driven simulation. |
| `targetcustomserial.h` | Custom target hook | Board-specific overrides (rare). |

Plus shared bits:

| File | Role |
|------|------|
| `frsky_crc.c`, `frsky_crc.h` | Shared CRC for FrSky family. |
| `pwm.c`, `pwm.h` | Legacy 8-channel PWM RX (one pin per channel). Mostly retired. |
| `rc_stats.c`, `rc_stats.h` | Frame-rate / link-quality stats common to all serial RXs. |
| `rx_bind.c`, `rx_bind.h` | "Bind" mode entry — for SPI RX especially, plus protocols that need a bind sequence. |
| `msp.c`, `msp.h` | MSP-as-RX — let an external app drive RC channels via MSP frames. Used for SITL + bench testing. |
| `msp_override.c`, `msp_override.h` | Override specific channels via MSP at runtime (for software-in-the-loop tuning). |

## SPI RX protocols

Here the FC has the RF chip soldered to its SPI bus and runs the entire radio link itself: bind sequences, frequency hopping, FEC, CRC, telemetry packet construction. Cheaper hardware (no separate RX module) but more firmware complexity.

Each chip family has its own driver pair plus per-protocol layers.

### CC2500 (TI 2.4 GHz)

| File | Protocol |
|------|----------|
| `cc2500_common.c`, `.h` | Common CC2500 driver (init, register access, hop tables). |
| `cc2500_frsky_d.c`, `.h` | FrSky D8 (D-series) RX. |
| `cc2500_frsky_x.c`, `.h` | FrSky D16 (X-series, ACCESS). |
| `cc2500_frsky_common.h`, `cc2500_frsky_shared.c`, `_shared.h` | FrSky shared helpers. |
| `cc2500_sfhss.c`, `.h` | Futaba SFHSS. |
| `cc2500_redpine.c`, `.h` | Red Pine (modified D8 by RedAcademy). |

### CYRF6936 (Cypress 2.4 GHz)

| File | Protocol |
|------|----------|
| `cyrf6936_spektrum.c`, `.h` | Spektrum DSM2/DSMX over CYRF6936. |

### A7105 (AMICCOM 2.4 GHz)

| File | Protocol |
|------|----------|
| `a7105_flysky.c`, `.h`, `a7105_flysky_defs.h` | Flysky AFHDS2A (i6, i10). |

### NRF24L01 (Nordic 2.4 GHz)

The classic toy-grade radio chip — supported for budget cheap-and-cheerful quads.

| File | Toy protocol |
|------|--------------|
| `nrf24_cx10.c`, `.h` | Cheerson CX-10. |
| `nrf24_h8_3d.c`, `.h` | Eachine H8 3D. |
| `nrf24_syma.c`, `.h` | Syma X5C and family. |
| `nrf24_v202.c`, `.h` | WLToys V202. |
| `nrf24_kn.c`, `.h` | KN-style (multiple cheap variants). |
| `nrf24_inav.c`, `.h` | INAV's custom NRF24 protocol — included for compat. |

### SX127x / SX1280 — ExpressLRS

| File | Role |
|------|------|
| `expresslrs.c`, `expresslrs.h`, `expresslrs_impl.h` | Top-level ExpressLRS module. State machine, hop sequencer, MSP-over-OTA tunnel, dyn-power, telemetry assembly. |
| `expresslrs_common.c`, `expresslrs_common.h` | Shared protocol constants (modulation tables, packet rates, FHSS sequences). |
| `expresslrs_telemetry.c`, `expresslrs_telemetry.h` | Telemetry sender (ELRS uses CRSF as its over-the-wire payload). |

ExpressLRS supports both 900 MHz (SX1276/77/78 via `drivers/rx/`) and 2.4 GHz (SX1280) chips with the same upper protocol stack.

## Transport abstraction — `rx_spi.c`

`rx_spi.c` and `rx_spi_common.c` sit between the SPI-RX protocol layer and the actual SPI bus driver. They provide:

- `rxSpiInit()` — bring up the RF chip's SPI peripheral with the right speed, mode, CS pin.
- `rxSpiWriteRegister()`, `rxSpiReadRegister()`, `rxSpiReadRegisterMulti()` — register R/W helpers.
- `rxSpiReadCommand()` — multi-byte command issue.
- `rxSpiSetTxRxMode()` — flip the chip between TX and RX modes.

Each chip-family driver (`cc2500_common.c`, `cyrf6936_spektrum.c`, `a7105_flysky.c`, …) uses these to do its register accesses, so adding support for a new chip-family means only the chip-family driver + its protocols change — the SPI transport stays the same.

`rx_spi_protocol_e` enum (in `rx_spi.h`) selects which SPI protocol is active at runtime:

```c
typedef enum {
    RX_SPI_NRF24_V202_250K = 0,
    RX_SPI_NRF24_SYMA_X,
    RX_SPI_NRF24_SYMA_X5C,
    RX_SPI_NRF24_CX10, RX_SPI_NRF24_CX10A,
    RX_SPI_NRF24_H8_3D,
    RX_SPI_NRF24_INAV,
    RX_SPI_NRF24_KN,
    RX_SPI_A7105_FLYSKY,
    RX_SPI_A7105_FLYSKY_2A,
    RX_SPI_CC2500_FRSKY_D,
    RX_SPI_CC2500_FRSKY_X,
    RX_SPI_CC2500_FRSKY_X_LBT,
    RX_SPI_CC2500_SFHSS,
    RX_SPI_CYRF6936_DSM,
    RX_SPI_CC2500_REDPINE,
    RX_SPI_EXPRESSLRS,
    RX_SPI_PROTOCOL_COUNT
} rx_spi_protocol_e;
```

User selects via `set rx_spi_protocol = EXPRESSLRS` (CLI) or via the Configurator drop-down.

## Bind mode

SPI-based RXs need a bind sequence to pair with the transmitter. `rx_bind.c` is the shared mechanism: enters a "BIND" state where the protocol exchanges identifying frames with the TX, captures the TX's ID/hop-pattern, and writes it to a persistent PG. On reboot the FC uses the bound ID to filter for that specific TX's packets.

Entry methods:

- `MSP_RX_BIND` — Configurator triggers it.
- CLI `bind_rx` command.
- Sticks combo at boot (Spektrum DSM style).
- Dedicated bind button on the FC (rare).

## Channel data flow

```
RF / wire frame
  ↓
protocol parser (sbus.c / crsf.c / expresslrs.c / ...)
  ↓
sbusChannelData[16] / crsfChannelData[16] / ...   (protocol-native order)
  ↓
rxRuntimeState.rcReadRawFn(state, channelIdx)     (lazy read from dispatcher)
  ↓
rx.c::readRxChannelsApplyRanges()
  - apply rxConfig()->rcmap[]                    (user reorder)
  - clamp to PWM_RANGE_MIN..MAX
  - apply rcSmoothing / per-channel rxfail behaviour
  ↓
rcData[ROLL/PITCH/YAW/THROTTLE/AUX1..]            (consumed by fc/rc.c)
```

`fc/rc.c::processRcCommand()` then takes `rcData[]` → smoothing → expo → rates → setpoint, as covered in [[05-flight-core-loop]].

## RSSI sources

`rxConfig()->rssi_src_frame_errors` and friends select where RSSI comes from:

- **Embedded in protocol frames** — CRSF, GHST, SBUS2, FPort, FrSky X, ExpressLRS, SRXL2. Best.
- **Dedicated RC channel** — protocols that don't have native RSSI but the user assigns AUX channel N to carry it.
- **ADC pin** — analog RSSI from an external receiver. Needs `USE_RX_RSSI_ADC` and the right pin in target.
- **Frame error rate** — fallback computed in `rx.c` (high RSSI when frames arrive on time, low when they're missed).

RSSI is exposed as `getRssi()` (0–1023) and consumed by OSD, telemetry, and beeper alarm.

## Failsafe

Three layers compose:

1. **Per-channel failsafe** (`rxFailsafeChannelConfiguration_t`) — for each channel, what to do when frames stop: AUTO (centre for ROLL/PITCH/YAW, low for THROTTLE), HOLD (last value), or SET (fixed value). Configured per channel.
2. **Stage 1** — Brief loss (< `failsafe_delay`). Outputs hold last/configured values, no flight-state change.
3. **Stage 2** — Sustained loss (≥ `failsafe_delay` and ≤ `failsafe_off_delay`). Configurable behaviour: `DROP` (disarm), `LAND` (lower throttle, level attitude), `GPS_RESCUE` (RTH).

Stage transitions are in `flight/failsafe.c`. Failsafe state changes set/clear `FAILSAFE_MODE` in `flightModeFlags` and update `armingDisableFlags` so re-arm requires explicit recovery.

## Where things live

| Concern | File |
|---------|------|
| Top-level RX dispatcher | `src/main/rx/rx.c`, `rx.h` |
| Channel data | `rcData[]` in `rx.c` (global) |
| Serial RX protocols | `src/main/rx/<protocol>.c` |
| SPI RX dispatcher | `src/main/rx/rx_spi.c`, `rx_spi_common.c` |
| Chip-family drivers | `cc2500_common.c`, `cyrf6936_spektrum.c`, `a7105_flysky.c`, nrf24_*.c |
| ExpressLRS | `expresslrs.c`, `expresslrs_common.c`, `expresslrs_telemetry.c` |
| Bind mode | `rx_bind.c` |
| Failsafe | `src/main/flight/failsafe.c`, `failsafe.h` |
| RC processing downstream | `src/main/fc/rc.c` |
| RX config struct | `src/main/pg/rx.h` (the PG_RX_CONFIG record) |
| RF chip drivers | `src/main/drivers/rx/` |

## See also

- [[CRSF Protocol]] for the most common modern RX wire format
- [[05-flight-core-loop]] for what happens with `rcData[]` after RX produces it
- [[07-hal-and-drivers]] for the RF chip drivers underneath
- `set rx_spi_protocol`, `bind_rx` CLI commands
