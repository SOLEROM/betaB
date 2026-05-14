---
type: architecture
status: stable
created: 2026-05-14
updated: 2026-05-14
tags: [osd, blackbox, telemetry, presentation]
source-commit: 6434dd725
---

# 10 — OSD, Blackbox, Telemetry

Three subsystems that share one property: they read flight state and ship it outward — to the pilot's goggles, to a log file, or back up the radio link. They never write to flight state.

## OSD — on-screen display elements

### Files

| File | Lines | Role |
|------|-------|------|
| `osd/osd.c` | ~1700 | Top-level OSD manager. Element render loop, profile switching, blink timer, stats screen, post-flight summary. |
| `osd/osd.h` | — | Public API (`osdInit()`, `osdUpdate()`, `osdResetStats()`). |
| `osd/osd_elements.c` | — | One render function per element (artificial horizon, speed, altitude, battery, GPS, RSSI, debug, timers, warnings, callsign, sticks, …). ~80 elements. |
| `osd/osd_elements.h` | — | Element ID enum (`OSD_ARTIFICIAL_HORIZON`, `OSD_VTX_CHANNEL`, …) plus per-element data structs. |
| `osd/osd_custom_text.c` | — | User-defined static text strings shown as elements. |
| `osd/osd_warnings.c` | — | Warning banner generator. Disarm-reason banners, runaway warning, ESC overheat, etc. |

### Rendering model

The OSD doesn't draw pixels — it writes characters into a virtual framebuffer (`displayPort_t` from `drivers/display.h`). Two flavours:

- **SD OSD**: 30 columns × 16 rows (PAL) / 13 rows (NTSC), MAX7456 character generator chip. Custom 16×18 px monochrome font with 256 glyphs.
- **HD OSD**: 50 cols × 18 rows (16:9) or 60 × 22 (Walksnail/HDZero). Rendered by the goggle's hardware via MSP-displayport.

`osdConfig()->osdProfileIndex` (1, 2, or 3) selects one of three profiles; each profile is an independent layout of which elements are visible and where.

### Element render loop

Once per OSD task tick (60 Hz default):

```c
for each element in OSD_ITEM_COUNT:
   if element disabled in current profile: skip
   if element has blink and blink-state = off: skip
   pos = osdConfig()->item_pos[profile][element]
   if pos == 0: skip   // 0 = invisible
   row = OSD_Y(pos), col = OSD_X(pos)
   osdElementsRender[element](row, col)
```

Each `osdElementsRender[]` function pulls live state (gyro, attitude, GPS, battery, RC) and formats a string at the given grid position. Special-cased elements (artificial horizon, sticks, crosshair) span multiple cells.

### Stats screen

After disarming, the OSD shows a post-flight summary screen: flight time, max speed, max altitude, max current, mAh consumed, blackbox status, disarm reason. Toggled visible/invisible per stat in `osdConfig()->enabled_stats`. Implementation in `osd.c::osdShowStats()`.

`statsConfig()->stats_min_armed_time` filters out brief test arms.

### Warnings

`osd_warnings.c` produces a priority-ordered queue of warning banners (one shown at a time):

```
GPS RESCUE UNAVAILABLE     (critical)
BATTERY CRITICAL
LOW RSSI
FAILSAFE
RUNAWAY
RPM FILTER...               (info)
CRASH FLIP MOTOR OFF
```

The active warning replaces the normal OSD bottom banner.

### OSD ↔ CMS coexistence

CMS owns the framebuffer while open. `osdUpdate()` returns early. On CMS exit, the OSD repaints from scratch.

### How it talks to hardware

`osdConfig()->displayPortDevice` picks the active transport: `MAX7456`, `OLED`, `MSP`, `FRSKYOSD`, `CRSF`, `HOTT`, `SRXL`. Each maps to a `displayport_*.c` driver in `io/` (see [[08-io-subsystems]]). On HD systems the MSP displayport sends 30/50/60-wide character frames over the OSD link to the goggle, which renders.

## Blackbox — flight data logger

### Files

| File | Lines | Role |
|------|-------|------|
| `blackbox/blackbox.c` | ~2300 | Main logger: state machine, frame composition, header writer, end-of-log finaliser. |
| `blackbox/blackbox.h` | — | Public API (`blackboxInit()`, `blackboxUpdate()`, `blackboxLogEvent()`). |
| `blackbox/blackbox_encoding.c`, `.h` | — | Variable-length integer encoding (signed/unsigned VLI), float-to-integer scaling. |
| `blackbox/blackbox_io.c`, `.h` | — | Backend abstraction: flash, SD card, serial. |
| `blackbox/blackbox_fielddefs.h` | — | Definitions of every loggable field + their inclusion conditions. |
| `blackbox/blackbox_virtual.c` | — | RAM-only backend used in SITL. |

### What gets logged

A blackbox log is a sequence of typed frames:

| Frame | When | Content |
|-------|------|---------|
| `H` header | Once at start | Firmware version, board, fields list, sampling rate, gain values |
| `I` intra | Every N frames (e.g. 32) | Full snapshot of all tracked fields |
| `P` predicted | Between I-frames | Delta-encoded difference from prediction (4–10× compression vs I) |
| `S` slow | ~10 Hz | Battery voltage, attitude angles, flight mode flags, failsafe phase |
| `G` GPS | On GPS update | Coords, altitude, satellites, speed, course, home distance |
| `H` GPS home | On home set | Home coords + altitude |
| `E` event | On significant change | Disarm reason, custom flag, autoTune step, etc. |

Field list is dynamic — depends on enabled features (e.g. `magADC` only logged if compass present, `motor[5]` only if 6+ motors). The list goes in the header so the decoder knows what to expect.

### Encoding

Two compression layers:

1. **Predictive coding.** For each P-frame field, write the value relative to a prediction:
   - `PREDICTOR_0` — value 0 (encode raw value, fallback)
   - `PREDICTOR_PREVIOUS` — last frame's value
   - `PREDICTOR_AVERAGE_2` — average of last two frames
   - `PREDICTOR_MOTOR_0` — motor[0]'s value (so motor[1..N] are coded as differences)
   - several more
2. **Variable-length integers.** Small values fit in 1 byte, larger ones up to 5. Signed values are zigzag-encoded.

Result: a 4 kHz, 16-field log fits in ~80–150 kB per minute on flash, ~300 kB per minute on SD.

### Backends

`blackbox_device_e`:

| Backend | Storage | Capacity | Notes |
|---------|---------|----------|-------|
| `BLACKBOX_DEVICE_FLASH` | On-board SPI flash chip | 8–256 MB typical | Circular buffer, one log per arming |
| `BLACKBOX_DEVICE_SDCARD` | SD card via SDIO/SPI | up to 32 GB FAT32 | One file per arming; faster, fault-tolerant |
| `BLACKBOX_DEVICE_SERIAL` | Stream to a UART | unlimited (over the wire) | Used by external loggers like the OpenLager |
| `BLACKBOX_DEVICE_NONE` | — | — | Disabled |

`blackbox_io.c` abstracts the backend: `blackboxDeviceWrite()`, `blackboxDeviceFlush()`, `blackboxDeviceReserveBufferSpace()`. Each backend has its own write strategy (flash needs page-aligned writes, SD does async DMA via `io/asyncfatfs/`).

### Sampling decimation

`blackboxConfig()->sample_rate` (1, 2, 4, 8, 16, 32) divides the PID loop rate. At 8 kHz PID, `sample_rate = 4` logs at 2 kHz. Higher sample rates produce bigger logs but better post-flight analysis (esp. for tuning gyro filters).

### Integration with the flight loop

`subTaskPidSubprocesses()` calls `blackboxUpdate(currentTimeUs)` after the mixer writes motors. The task `TASK_BLACKBOX` runs at PID rate but internally decimates — see `blackbox.c::blackboxAdvanceIterationTimers()`. The actual frame compose + write happens on the PID task path, so there's never a context-switch cost.

### Log retrieval

- Onboard flash → USB MSC mode (FC mounts the flash as a USB drive) → drag log file off.
- SD card → physically remove + read on PC.
- Serial → external logger captures live.

Analyse with [Betaflight Blackbox Explorer](https://github.com/betaflight/blackbox-explorer) (Electron app). Decodes the binary, plots traces (gyro, setpoint, PID output, motor command, battery, attitude). Indispensable for tuning.

For the file format details see [[Blackbox Format]].

## Telemetry — downlink to transmitter

### Files

| File | Role |
|------|------|
| `telemetry/telemetry.c`, `.h` | Top-level dispatcher: init + per-frame processing for each enabled protocol. |
| `telemetry/crsf.c`, `.h` | CRSF (Crossfire / ExpressLRS). Most common modern protocol. |
| `telemetry/smartport.c`, `.h` | FrSky SmartPort. Half-duplex on inverted UART. |
| `telemetry/frsky_hub.c`, `.h` | FrSky Hub — legacy serial telemetry (D-series receivers). |
| `telemetry/ghst.c`, `.h` | Ghost (ImmersionRC GHST protocol). |
| `telemetry/hott.c`, `.h` | Graupner HoTT. |
| `telemetry/ibus.c`, `.h` | Flysky iBus telemetry. |
| `telemetry/srxl.c`, `.h` | Spektrum SRXL (legacy). |
| `telemetry/jetiexbus.c`, `.h` | Jeti ExBus. |
| `telemetry/ltm.c`, `.h` | LTM (Lightweight Telemetry) — popular with ground stations. |
| `telemetry/mavlink.c`, `.h` | MAVLink — used by autopilot integrations (SITL with QGroundControl etc.). |
| `telemetry/msp_shared.c`, `.h` | MSP-over-telemetry tunnelling — lets Configurator run over the radio link. |

Plus `telemetry/sensors.c`, `.h` and helpers like `telemetry/sensors.h` for shared "what does this protocol need" plumbing.

### Selection model

A telemetry protocol is bound to a UART by setting that port's `FUNCTION_TELEMETRY_<PROTOCOL>` in `serialConfig`. Exactly one telemetry protocol is active at a time per port; multiple ports could each carry a different protocol but in practice one is normal.

Some protocols can co-exist on a port with their RX counterpart via half-duplex sharing — SBUS + SmartPort, CRSF (always bidirectional), FPort (combined RX + telemetry).

### Dispatcher

`telemetry.c::telemetryProcess()` runs at `TASK_TELEMETRY` rate (250 Hz) and invokes each enabled protocol's handler:

```c
void telemetryProcess(timeUs_t currentTimeUs) {
#ifdef USE_TELEMETRY_FRSKY_HUB
    handleFrSkyHubTelemetry(currentTimeUs);
#endif
#ifdef USE_TELEMETRY_HOTT
    handleHoTTTelemetry(currentTimeUs);
#endif
#ifdef USE_TELEMETRY_SMARTPORT
    handleSmartPortTelemetry();
#endif
    // ...
}
```

Protocols self-rate-limit; CRSF for instance schedules different "sensor frames" at different cadences (battery 10 Hz, GPS 5 Hz, attitude 10 Hz).

### What gets sent

The intersection of "what the FC knows" and "what the protocol supports". Typical fields:

- **Battery** — voltage, current, mAh consumed, cell count, remaining percentage.
- **GPS** — lat, lon, altitude, sats, speed, heading.
- **Attitude** — roll, pitch, yaw angles.
- **Vario** — vertical speed (baro-derived).
- **Home** — distance + bearing.
- **Link** — RSSI, link quality, SNR (where the radio reports them).
- **Flight state** — armed, flight mode, failsafe.
- **Temperatures** — ESC temps (DShot telemetry), board temp.

CRSF carries the richest set and updates fastest.

### MSP-over-CRSF (and friends)

Some link technologies (CRSF, GHST) tunnel MSP through their telemetry channel: the transmitter forwards MSP frames between Configurator-on-laptop (via a USB joystick or BT) and the FC. This lets you live-tune over the air. Implementation: `telemetry/msp_shared.c` plus `rx/crsf.c::crsfRxSendTelemetryData()` for the wire side. Configurator must run in "MSP over radio" mode.

### Telemetry config

`telemetryConfig_t` (PG) holds:

- `telemetry_inverted` — invert UART polarity for SmartPort.
- `halfDuplex` — half-duplex mode for protocols that need it.
- `gpsNoFixLatitude/Longitude` — what to send before GPS lock.
- `frsky_unit` — metric vs imperial.
- `frsky_default_lat/lon` — fake home coords (for GPS-less builds).
- `report_cell_voltage` — sum vs per-cell average.
- `mavlink_mah_as_heading_divisor` — bizarre legacy bridging knob.
- `pidValuesAsTelemetry` — broadcast PID values for live OSD displays.

## Putting it together — info flow off the FC

```
flight loop produces in-memory state:
   gyro[XYZ]           → gyroADCf[]
   attitude.values     ← imu.c
   GPS_coord           ← gps.c
   battery state       ← battery.c
   motor[]             ← mixer.c
   pidData[].Sum       ← pid.c
   flightModeFlags     ← rc_modes.c
   armingFlags         ← runtime_config.c
                       ↓
                ┌──────┴───────┐
                ▼              ▼              ▼
              OSD          Blackbox       Telemetry
              osd.c        blackbox.c     telemetry.c
              every 60Hz   every Nth      every 250Hz
              ↓            PID cycle      tick
        framebuffer        ↓              ↓
              ↓            flash/SD/      protocol
        displayport_*      serial         frame
              ↓            file           encode
        MAX7456 SPI                       ↓
        or MSP→DJI                        UART/USB
        OLED, etc.                        to radio link
```

The three pipelines read but never write the in-memory state — flight control is unaffected by what's painted, logged, or beamed out.

## See also

- [[09-msp-cli-cms]] for MSP (separate from telemetry: MSP is the config protocol, telemetry is the downlink).
- [[Blackbox Format]] for binary log format reverse engineering.
- [[OSD Font Format]] for MAX7456 font tooling.
- [[CRSF Protocol]] for the wire format of the most common telemetry/RX protocol.
- [[08-io-subsystems]] for the `displayport_*` transport drivers.
