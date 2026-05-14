---
noteId: "8d4ecad04f8211f194a2c3b1eecd91b7"
tags: []

---

# MSP Stack

## Overview

`bt_app/msp/` is a from-scratch Python implementation of the Betaflight
**MultiWii Serial Protocol** (v1 and v2), a pluggable transport layer
(TCP / UDP / Serial), a high-level client with typed helpers
(`read_altitude`, `read_state`, `send_raw_rc`, …), and a **scheduled
command dispatcher** that owns a background thread and serializes RC/state
traffic to the FC. Every other module that talks to the FC goes through
this stack — there is no direct socket access elsewhere in `bt_app`.

## Key Decisions

- **Three layers, clean Protocol boundary.**
  `MspCodec` ↔ `ByteTransport` ↔ `BetaflightMspClient`. The transport is a
  duck-typed `Protocol` with `read(size, timeout)` / `write(data)`
  (`protocol.py:19-25`), so swapping TCP for serial is a constructor change.
- **Auto-select MSP v1 vs v2.** `encode_request` picks v2 only when
  command > 255 or payload > 255 bytes (`protocol.py:46-55`). All current
  callers stay on v1; v2 is wired but cold.
- **Resync by scanning for `$` byte.** `read_frame` discards bytes until
  it sees the magic header, then dispatches on `M`/`X` for v1/v2
  (`protocol.py:99-113`). Survives spurious bytes mid-stream — important
  because Betaflight emits async CLI bytes on the same UART.
- **CRC8 DVB-S2 for v2 only.** v1 uses an XOR checksum; v2 uses
  CRC-8-DVB-S2 (`protocol.py:185-194`).
- **MSP_SET_RAW_RC is fire-and-forget.** `send_raw_rc` writes the frame
  and tries a 5 ms decode; `TimeoutError` is swallowed (`bt_v2.py:126-136`).
  This is correct because RC is high-rate and the FC ACK is optional.
- **Dispatcher = priority queue keyed by `(run_at, seq, token)`** with a
  **per-key token** mechanism that lets a new `set_rc` cancel the
  recurring loop of an older one (`command_dispatcher.py:165-201`,
  `:298-301`). This means the latest RC commander wins automatically.
- **Scheduled commands self-reschedule.** When a command's
  `repeat_interval_s` is set, the dispatcher resubmits it after each
  execution iff its token is still active (`:263-270`).
- **Built-in commands.** `RawRcCommand`, `ReadStateCommand`,
  `ReadAltitudeCommand`, `HoverAtAltitudeCommand` (legacy in-dispatcher PID),
  `FunctionCommand`, `ScheduledCommand` wrapper.
- **`parse_status_ex` decodes 28 arming-disable flag bits** by name
  (`bt_v2.py:47-77`, `:190-241`). Behavior tree's `WaitArmable` uses
  this to gate takeoff.

## Constraints

- 8 RC channels, range-validated 800–2200 µs (`normalize_channels`,
  `command_dispatcher.py:303-312`). Outside that throws — controllers
  must clamp before submitting.
- Channel-order convention is fixed: `(ROLL, PITCH, THROTTLE, YAW, AUX1,
  AUX2, AUX3, AUX4)` (`bt_v2.RCChannel`). A parallel `RCChannel_alias`
  re-names AUX1=ARM, AUX2=ANGLE — must match Betaflight's mode switches.
- Dispatcher is single-threaded; long-running commands stall everything.
  Currently safe because all commands are short MSP round-trips.
- `read_state` calls `MSP_STATUS_EX` (cmd 150), which Betaflight only
  emits when configured — older firmwares would 404.
- Error semantics: `_handle_error` re-raises unless `on_error` is set
  (`:292-296`). In `main.py` we set a logger callback, so errors are
  swallowed — be aware.

## Open Questions

- `my_dispatcher.py` exists alongside `command_dispatcher.py` — an
  older parallel implementation? Unused by `main.py`. Delete on rebuild?
- No reconnect logic on TCP drop: if Betaflight dies, the dispatcher
  thread crashes on the next `recv`. Should reconnect or escalate.
- v2 path is untested; first command > 255 will exercise unverified code.
- Read commands are scheduled at fixed intervals (state @ 1 Hz, altitude
  @ 10 Hz) but RC at 50 Hz — when the FC TCP socket is slow this leads
  to head-of-line blocking. No timeout/backpressure policy.
