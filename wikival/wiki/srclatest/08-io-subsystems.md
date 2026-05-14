---
type: architecture
status: stable
created: 2026-05-14
updated: 2026-05-14
tags: [io, serial, vtx, gps, ledstrip, beeper, dronecan]
source-commit: 6434dd725
---

# 08 — I/O & Feature Subsystems (`src/main/io/`)

`io/` is the **application layer of the firmware**. It sits above `drivers/` (HAL) but below the presentation surfaces (MSP/CLI/CMS/OSD). Each file owns one user-facing feature — a VTX protocol, the LED strip animator, the GPS parser, the BLHeli pass-through bridge — and is responsible for both wiring up to its driver(s) and exposing config knobs via PG records.

Roughly 80 files. Grouped here by function.

## Serial port dispatcher

| File | Role |
|------|------|
| `serial.c`, `serial.h` | Top-level serial-port manager. Maps `SERIAL_PORT_FUNCTION_*` (MSP, GPS, telemetry, RX, blackbox, …) to physical UARTs via the `serialConfig` PG. Opens ports lazily as features request them. |
| `serial_resource.c` | Port-resource accounting (which port is owned by which function). |

The "function" abstraction matters because most ports can serve any role — Configurator might be on UART1, GPS on UART2, blackbox on UART3 — and the FC has to multiplex requests. Functions can be shared on a single port (`PORTSHARING_SHARED`) when protocols are compatible (e.g. SBUS RX + SmartPort telemetry on the same UART via inverter toggle).

### BLHeli 4-way / ESC programming

| File | Role |
|------|------|
| `serial_4way.c`, `serial_4way.h`, `serial_4way_impl.h` | The "4-way" interface that Configurator's BLHeli pass-through uses. Routes serial traffic directly to one ESC's UART line. |
| `serial_4way_avrootloader.c`, `_avrootloader.h` | BLHeli (AVR-based) bootloader protocol. |
| `serial_4way_stk500v2.c`, `_stk500v2.h` | STK500 v2 bootloader (used by some KISS/BLHeli32 ESCs). |

## VTX (video transmitter) control

VTX protocols are bidirectional, talking to a separate VTX chip over UART. Each protocol has its own file pair plus a low-level register driver in `drivers/vtx_*`.

| File | Role |
|------|------|
| `vtx.c`, `vtx.h` | Top-level VTX manager. Owns the active protocol pointer, band/channel/power state. |
| `vtx_control.c`, `vtx_control.h` | Pilot interface — applies CMS / RC-mode-driven band/channel/power changes. |
| `vtx_table.c` | Band/channel frequency lookup table (custom band support). |
| `vtx_msp.c`, `vtx_msp.h` | VTX-over-MSP — for DJI HD systems that proxy VTX commands through the OSD link. |
| `vtx_smartaudio.c`, `vtx_smartaudio.h` | TBS SmartAudio (single-wire UART). |
| `smartaudio_protocol.c`, `smartaudio_protocol.h` | SmartAudio frame encode/decode helpers. |
| `vtx_tramp.c`, `vtx_tramp.h` | ImmersionRC Tramp protocol. |
| `tramp_protocol.c`, `tramp_protocol.h` | Tramp frame helpers. |
| `vtx_rtc6705.c`, `vtx_rtc6705.h` | Direct RTC6705 SPI register control (legacy on-board VTX chips). |

## GPS

| File | Role |
|------|------|
| `gps.c`, `gps.h` | Top-level GPS parser. Supports NMEA, UBX (u-blox), and a UBX-only fast init mode. Maintains `GPS_coord`, `GPS_numSat`, `GPS_hdop`, `GPS_speed`, `GPS_ground_course`, `GPS_altitude`. |
| `gps_virtual.c`, `gps_virtual.h` | Simulated GPS for SITL. Feeds fake fix from UDP link or static config. |

## LED strip

| File | Role |
|------|------|
| `ledstrip.c`, `ledstrip.h` | WS2811/SK6812/APA102 animator. Per-LED config: function (mode, battery, beacon, …), colour, direction. Many "modes" overlay (flight mode indicators, battery gauge, blink-on-arming, racing wing colours, etc.). Driven by `drivers/light_ws2811strip.c`. |

## Beeper / audio

| File | Role |
|------|------|
| `beeper.c`, `beeper.h` | Multi-source beeper. Queues tones by priority. Sources include `BEEPER_GYRO_CALIBRATED`, `BEEPER_RX_LOST`, `BEEPER_ARMING`, `BEEPER_BAT_LOW`, etc. |
| `pidaudio.c`, `pidaudio.h` | Audio rendering of PID activity for debug — turns gyro/setpoint difference into audible tone. Optional / experimental. |

## OSD plumbing (transports, not elements)

`io/` owns the *transport* of OSD data to whichever display tech is fitted. Each `displayport_*.c` implements the abstract `display_t` API from `drivers/display.h`:

| File | Role |
|------|------|
| `displayport_max7456.c` | MAX7456 analog OSD chip (CCD overlay on FPV video signal). |
| `displayport_oled.c` | On-board I²C OLED (mostly bench / dev kit usage). |
| `displayport_msp.c` | OSD over MSP — for HD systems (DJI, Walksnail, HDZero, Avatar) where the goggles render the OSD. |
| `displayport_crsf.c` | OSD over CRSF link — TBS Tango / Crossfire display passthrough. |
| `displayport_hott.c` | OSD over HoTT telemetry — Graupner transmitters. |
| `displayport_frsky_osd.c` | FrSky pixel OSD (FrSky's own pixel-grid display). |
| `displayport_srxl.c` | OSD over SRXL2 (Spektrum) telemetry. |
| `displayport_fb_osd.c` | Generic framebuffer OSD (pixel-based, no character grid). |
| `frsky_osd.c`, `frsky_osd.h` | FrSky OSD-specific pixel & character routines. |

`dashboard.c`, `dashboard.h` is a legacy character LCD dashboard (mostly retired but still compilable for boards with onboard OLEDs).

The actual element rendering (artificial horizon, battery, speed, etc.) lives in `src/main/osd/osd_elements.c` — see [[10-osd-blackbox-telemetry]]. The displayport drivers only carry the resulting framebuffer to hardware.

## RunCam (camera control)

| File | Role |
|------|------|
| `rcdevice.c`, `rcdevice.h` | RunCam Device protocol over UART. Sends button presses to RunCam cameras. |
| `rcdevice_cam.c`, `rcdevice_cam.h` | High-level camera control (start/stop record, OSD overlay toggle). RC mode boxes `BOXCAMERA1/2/3` map to this. |

## Spektrum specifics

| File | Role |
|------|------|
| `spektrum_rssi.c`, `spektrum_rssi.h` | Spektrum-format RSSI extraction (the `srxl2` and `spektrum` RX paths have a quirk for embedded RSSI). |
| `spektrum_vtx_control.c`, `spektrum_vtx_control.h` | Spektrum's VTX control protocol (uses dedicated AUX channels with magic ranges). |

## Gimbal

| File | Role |
|------|------|
| `gimbal.h`, `gimbal_control.c`, `gimbal_control.h` | 3-axis gimbal control. Outputs pitch/roll/yaw to dedicated servo channels. Configurable as stabilised or pilot-controlled. |

## Programmable I/O ("pinio")

| File | Role |
|------|------|
| `piniobox.c`, `piniobox.h` | Programmable I/O channels. The user maps "if BOXUSER1 is active, set PINIO_1 high". Used for: VTX power switching, camera switching, GoPro trigger, auxiliary LED, fan control. Number of channels and pin mapping is per-board. |

## Storage

| File | Role |
|------|------|
| `flashfs.c`, `flashfs.h` | Flash file system used as a blackbox backend. Single-file circular log on raw flash. |
| `asyncfatfs/` (subdir) | Async FatFS wrapper for SD card blackbox. Non-blocking writes via DMA. |

## Transponder

| File | Role |
|------|------|
| `transponder_ir.c`, `transponder_ir.h` | IR transponder for FPV race timing systems (TBS, IROC, Trackside). Uses `drivers/transponder_ir_*` to time-modulate an IR LED. |

## USB classes

| File | Role |
|------|------|
| `usb_cdc_hid.c`, `usb_cdc_hid.h` | USB CDC + HID composite device. Lets the FC appear as a joystick to a PC (for FPV simulators). |
| `usb_msc.c`, `usb_msc.h` | USB Mass Storage Class — exposes the SD card / onboard flash to a host PC for log retrieval. |

## Status indicator

| File | Role |
|------|------|
| `statusindicator.c`, `statusindicator.h` | Drives the onboard status LED based on system state (boot, armed, error, calibrating, beacon). |

## DroneCAN

| Subdir | Role |
|--------|------|
| `dronecan/` | DroneCAN v1 node implementation. Uses `lib/main/dronecan/libcanard` as the transport library and `drivers/can/` for the CAN peripheral. Publishes: actuator status, RC input, GPS fix, sensor data. Subscribes: ESC command, RC override. Brings up only when a target's `mk/<MCU>.mk` opts in via `LIB_SUBMODULES += $(DRONECAN_LIB_DIR)`. |

## How a feature crosses the layers — VTX example

```
User toggles VTX power mode via OSD menu
└→ cms/cms_menu_vtx_smartaudio.c (Layer 5)
   └ writes vtxSettingsConfigMutable()->power
└→ vtxCommonProcess() runs at TASK_VTXCTRL rate (Layer 4)
   └ io/vtx.c sees power change
   └ calls vtxTable* lookup for the active protocol
   └ io/vtx_smartaudio.c::vtxSmartAudioSetPowerByIndex(power)
   └ encodes a SmartAudio frame
   └ writes via serialWrite() to the assigned UART (Layer 4)
└→ drivers/serial.h vTable → platform/STM32/serial_uart_stm32h7xx.c (Layer 1+2)
   └ HAL_UART_Transmit_DMA on UART4
└→ physical wire to VTX chip
└→ VTX returns ACK frame
└→ UART RX IRQ → buffer → vtxSmartAudioProcess() picks it up next cycle
└→ vtxCommonGetStatus() now reports power=400mW
└→ OSD element renders the updated power
```

Same shape applies to GPS, blackbox, telemetry, MSP. The pattern is uniform across the codebase.

## Task wiring

`io/` features attach to the scheduler via the task table (`fc/tasks.c`). Representative entries:

```c
[TASK_SERIAL]           = DEFINE_TASK("SERIAL",     NULL, NULL, taskHandleSerial,    TASK_PERIOD_HZ(100), TASK_PRIORITY_LOW),
[TASK_BATTERY_VOLTAGE]  = DEFINE_TASK("BATTERY_VOLTAGE", NULL, NULL, batteryUpdateVoltage, TASK_PERIOD_HZ(50), TASK_PRIORITY_MEDIUM),
[TASK_BATTERY_CURRENT]  = DEFINE_TASK("BATTERY_CURRENT", NULL, NULL, batteryUpdateCurrentMeter, TASK_PERIOD_HZ(50), TASK_PRIORITY_MEDIUM),
[TASK_BATTERY_ALERTS]   = DEFINE_TASK("BATTERY_ALERTS",  NULL, NULL, batteryUpdateAlarms, TASK_PERIOD_HZ(5),  TASK_PRIORITY_MEDIUM),
[TASK_BEEPER]           = DEFINE_TASK("BEEPER",     NULL, NULL, beeperUpdate,        TASK_PERIOD_HZ(100), TASK_PRIORITY_LOW),
[TASK_GPS]              = DEFINE_TASK("GPS",        NULL, NULL, gpsUpdate,           TASK_PERIOD_HZ(100), TASK_PRIORITY_MEDIUM),
[TASK_TELEMETRY]        = DEFINE_TASK("TELEMETRY",  NULL, NULL, taskTelemetry,       TASK_PERIOD_HZ(250), TASK_PRIORITY_LOW),
[TASK_LEDSTRIP]         = DEFINE_TASK("LEDSTRIP",   NULL, NULL, ledStripUpdate,      TASK_PERIOD_HZ(100), TASK_PRIORITY_LOW),
[TASK_TRANSPONDER]      = DEFINE_TASK("TRANSPONDER",NULL, NULL, transponderUpdate,   TASK_PERIOD_HZ(250), TASK_PRIORITY_LOW),
[TASK_VTXCTRL]          = DEFINE_TASK("VTXCTRL",    NULL, NULL, vtxUpdate,           TASK_PERIOD_HZ(5),   TASK_PRIORITY_IDLE),
[TASK_OSD]              = DEFINE_TASK("OSD",        NULL, NULL, osdUpdate,           TASK_PERIOD_HZ(60),  TASK_PRIORITY_LOW),
[TASK_CMS]              = DEFINE_TASK("CMS",        NULL, NULL, cmsHandler,          TASK_PERIOD_HZ(60),  TASK_PRIORITY_LOW),
[TASK_BLACKBOX]         = DEFINE_TASK("BLACKBOX",   NULL, NULL, taskBlackbox,        TASK_PERIOD_HZ(8000), TASK_PRIORITY_LOW),
[TASK_PINIOBOX]         = DEFINE_TASK("PINIOBOX",   NULL, NULL, pinioBoxUpdate,      TASK_PERIOD_HZ(20),  TASK_PRIORITY_LOW),
```

Notice the wide range of rates (5 Hz for VTX control, 250 Hz for transponder, 8 kHz for blackbox sampling) and that everything is **at most `TASK_PRIORITY_MEDIUM`** — never `REALTIME`. None of these may starve the flight loop.

`taskBlackbox` is special-cased: it's not actually called every 125 µs even though its period says so — the logger internally decimates to the configured `blackbox_sample_rate`. The high "desired period" just makes the scheduler keep visiting it frequently.

## See also

- Layer 5 surfaces (CLI, CMS, MSP) are covered in [[09-msp-cli-cms]].
- OSD elements (the things rendered on screen) are in [[10-osd-blackbox-telemetry]].
- RX protocols (also a "feature" but big enough for its own page) are in [[11-rx-subsystem]].
- The PG records these features read live in `src/main/pg/` — see [[12-config-and-pg]].
