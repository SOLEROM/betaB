---
type: architecture
status: stable
created: 2026-05-14
updated: 2026-05-14
tags: [msp, cli, cms, configurator, presentation]
source-commit: 6434dd725
---

# 09 — MSP, CLI, CMS — The three config interfaces

Betaflight exposes three independent interfaces for inspecting and changing configuration:

| Interface | Used by | Transport | Source dir |
|-----------|---------|-----------|------------|
| **MSP** | Betaflight Configurator (web/electron), Blackbox Explorer, any custom tool | UART / USB CDC / MSP-over-CRSF | `src/main/msp/` |
| **CLI** | Humans typing commands | UART / USB CDC (any port set to MSP also accepts CLI) | `src/main/cli/` |
| **CMS** | Pilot via RC sticks looking at OSD | RC channels + OSD framebuffer | `src/main/cms/` |

All three converge on the same in-memory Parameter Groups (`src/main/pg/*`). See [[12-config-and-pg]] for the PG system itself — this page covers the three presentation surfaces and how they hook in.

## MSP — the wire protocol

Already covered in detail at [[MSP Protocol]] and [[MSP Commands Reference]] (the protocol-side write-ups). This section focuses on the **source layout** and **dispatcher architecture**.

### Files

| File | Lines | Role |
|------|-------|------|
| `msp/msp.c` | ~4800 | The big command dispatcher: `mspFcProcessCommand()` + `mspFcProcessReply()` switch tables |
| `msp/msp.h` | — | Public API (`mspDescriptorAlloc()`, `mspWriteResponseToStream()`) |
| `msp/msp_protocol.h` | ~700 | Opcode constants (MSP v1 protocol — MultiWii-compatible subset) |
| `msp/msp_protocol_v2_betaflight.h` | — | Betaflight-specific MSPv2 opcodes |
| `msp/msp_protocol_v2_common.h` | — | Shared MSPv2 opcodes (used by INAV too) |
| `msp/msp_serial.c` | ~700 | Wire framing — MSP v1 ASCII header, MSP v2 binary header, CRC |
| `msp/msp_serial.h` | — | `mspSerialAllocatePorts()`, `mspSerialReleasePortIfAllocated()`, frame state machine |
| `msp/msp_box.c` | ~400 | Mode-box ↔ AUX activation mapping (the table behind `MSP_MODE_RANGES`) |
| `msp/msp_build_info.c` | — | `MSP_BUILD_INFO` opcode handler. Reports firmware build date, options, debug build flag. |

### Dispatcher pattern

`msp.c` is a giant switch on `cmdMSP`. Each opcode handler:

1. Reads payload bytes from the incoming buffer via `sbufRead*()`.
2. Validates parameters.
3. Either writes a reply via `sbufWrite*()` (queries) or mutates a PG struct (`MSP_SET_*`).
4. Returns `MSP_RESULT_ACK`, `MSP_RESULT_ERROR`, or `MSP_RESULT_NO_REPLY`.

The dispatcher uses **descriptors** to track who's talking on which port:

```c
mspDescriptor_t mspDescriptorAlloc(void);
void mspArmingDisableByDescriptor(mspDescriptor_t descriptor);
void mspArmingEnableByDescriptor(mspDescriptor_t descriptor);
```

Each MSP port (UART1, USB, CRSF tunnel) gets a unique descriptor. When Configurator opens MSP, it sets `ARMING_DISABLED_MSP` for its descriptor; only that descriptor can clear it. This lets multiple MSP clients connect (e.g. Configurator + blackbox tool) without one clobbering the other's arming state.

### Wire framing

Two coexisting framings:

- **MSPv1**: `$M<` (request) or `$M>` (reply), size byte, command byte, payload, XOR checksum. Limit 255-byte payload. ASCII-style header for easy debugging.
- **MSPv2**: `$X<` / `$X>`, flags, 16-bit command, 16-bit size, payload, CRC8/DVB-S2. Up to 64 KB payload. Required for opcodes ≥ 255.

The receiver `mspSerialProcess()` is a state machine over each port's RX byte stream. Detection of `$M` vs `$X` selects framing automatically. CRC mismatch silently drops the frame.

### What MSP carries

- **Read**: firmware version, sensor data (gyro/acc/attitude/baro/GPS), motor outputs, RC channels, settings, profiles, blackbox status, OSD config, modes, box ranges, build info.
- **Write (`MSP_SET_*`)**: all the same as read, plus setpoint overrides for testing, motor outputs (only when disarmed), motor reorder, RC tuning, profiles, OSD layout.
- **Special**: `MSP_EEPROM_WRITE` (commit modified PGs to flash), `MSP_REBOOT` (reboot into normal/DFU/bootloader/MSC), `MSP_RX_BIND` (enter SPI RX bind mode), `MSP_DEBUG_DATA` (debug pin telemetry).
- **Tunnelled**: `MSP_DISPLAYPORT` carries OSD framebuffer cells for HD systems. `MSP_VTX_CONFIG` etc. let DJI proxy VTX commands through the OSD link.

Configurator's typical sequence on connect: `MSP_API_VERSION` → `MSP_FC_VARIANT` → `MSP_FC_VERSION` → `MSP_BOARD_INFO` → `MSP_BUILD_INFO` → ... and only then does it issue setting reads.

### Where MSP plugs into the scheduler

`TASK_SERIAL` runs `taskHandleSerial()` (in `io/serial.c`) at 100 Hz. That function asks every active MSP port: "any pending frames?" If yes, it parses one frame and calls `mspFcProcessCommand()`. Replies are queued via `mspSerialPush()` and drained by the same task on subsequent ticks.

## CLI — terminal interface

### Files

| File | Lines | Role |
|------|-------|------|
| `cli/cli.c` | ~8600 | Command parser + every command's handler (`cliSet`, `cliGet`, `cliDump`, `cliFlashRead`, `cliResource`, `cliDshotProg`, …) |
| `cli/cli.h` | — | Public API (`cliEnter()`, `cliProcess()`) |
| `cli/settings.c` | ~2100 | **The settings table.** One entry per CLI setting. |
| `cli/cli_debug_print.h` | — | Debug print helpers |

### The settings table

`settings.c` is one giant array of `settingTable_t` entries. Each entry:

```c
{
    "altitude_lpf",                                   // setting name
    VAR_UINT16 | MASTER_VALUE,                        // type + scope
    .config.minmaxUnsigned = { 10, 1000 },            // bounds
    PG_POSITION,                                      // which PG owns the field
    offsetof(positionConfig_t, altitude_lpf)          // byte offset within that PG
}
```

This single record tells the CLI everything it needs to:

- Show `altitude_lpf` in `dump` output.
- Validate `set altitude_lpf=200` against `[10, 1000]`.
- Find the PG (PG_POSITION) and write the value at the right offset.
- Mark that PG dirty so `save` flushes it.

Setting types: `VAR_UINT8 | VAR_INT8 | VAR_UINT16 | VAR_INT16 | VAR_UINT32 | VAR_INT32`. Plus modifiers: `MASTER_VALUE` (single global), `PROFILE_VALUE` (per PID profile), `CONTROL_RATE_VALUE` (per rate profile). Plus mode flags: `MODE_DIRECT` (simple range), `MODE_LOOKUP` (enum-indexed table), `MODE_ARRAY` (fixed-size array), `MODE_BITSET` (bit position in a mask), `MODE_STRING` (string), `MODE_LOOKUP_HEX` (hex lookup), `MODE_DEFAULT_ANYWAY` (always shown in dump).

### Commands

The CLI dispatcher (`cli.c::cliProcess()`) tokenises a line and looks it up in a `clicmd_t` table. Standard commands:

```
get  set  status  version  tasks  resource  dma  timer  serial
profile  rateprofile  defaults  save  exit  reboot  bl  msc
diff  dump  feature  mixer  motor  led  color  mode_color
aux  map  serial_passthrough  smix  rxrange  led
playsound  beeper  vtx  vtx_info  pinio_test
flash_read  flash_write  flash_erase  flash_info
sd_info  dfu  bootloader_flash  bl
mcu_id  beacon  rxfailsafe  feature  vtxtable
dshotprog  escprog  msc  display_name
```

Each maps to a `cliXxx()` function in `cli.c`. The whole file is structurally simple — 80% is "parse args, call setting/PG accessors, format output".

`cli_debug_print.h` defines `cliPrintf` / `cliPrintLine` / `cliPrintError` etc. — all output funnels through the CLI's output buffer rather than printing directly to serial.

### `diff` and `dump`

`dump` prints the entire setting table as `set name=value` commands.
`diff` prints only the settings whose current value differs from the PG's default.

This is how the user backs up a config — `diff` output is human-readable, machine-applicable, and version-stable across firmware updates (because settings have stable names while their offsets in PG structs can shift).

`defaults nosave` reverts everything to PG defaults without writing flash; `defaults` does the same and saves.

### CLI ↔ MSP relationship

CLI and MSP are **independent paths to the same PGs**. MSP doesn't go through `settings.c` — it writes PG fields directly in its opcode handlers. The settings table is purely for CLI access.

This decoupling means: adding a new MSP opcode is independent of adding it to the CLI, and vice-versa. Some advanced features only exist over MSP (e.g. full OSD config blob) because the CLI representation would be ugly. Some exist only via CLI (`resource`, `timer` — bespoke output formats).

CLI mode is entered from MSP via `MSP_SET_CLI_MODE`. Once in CLI mode, the port speaks ASCII; an `exit`/`save` returns it to MSP framing.

## CMS — On-screen menu via RC sticks

### Files

`src/main/cms/` has ~20 files. Top-level + one per menu:

| File | Role |
|------|------|
| `cms.c`, `cms.h` | Top-level menu driver. Walks the menu tree, handles RC input, paints the active menu onto the OSD. |
| `cms_types.h` | Item-kind enums (`OME_INT8`, `OME_UINT16`, `OME_BOOL`, `OME_TAB`, `OME_FUNCTION`, …). |
| `cms_menu_main.c` | Root menu — links to all submenus. |
| `cms_menu_imu.c` | PID/rate tuning. The dominant CMS use. |
| `cms_menu_osd.c` | OSD layout (which elements visible, where). |
| `cms_menu_misc.c` | Misc settings (gyro, accelerometer trim, etc.). |
| `cms_menu_failsafe.c` | Failsafe configuration. |
| `cms_menu_blackbox.c` | Blackbox device/rate selection. |
| `cms_menu_firmware.c` | Firmware info display (version, build options). |
| `cms_menu_ledstrip.c` | LED strip animation tweaks. |
| `cms_menu_gps_rescue.c`, `cms_menu_gps.c` | GPS rescue parameters; GPS status. |
| `cms_menu_vtx_smartaudio.c`, `_tramp.c`, `_rtc6705.c`, `_msp.c`, `_common.c` | VTX control submenus per protocol. |
| `cms_menu_quick.c` | Fast-access menu for in-flight tweaks. |
| `cms_menu_saveexit.c` | Save / Save+Reboot / Exit-without-saving handlers. |
| `cms_menu_main.c` (top) | The root level. |

### Menu data structure

Each menu is a `CMS_Menu` struct holding a static array of `OSD_Entry`:

```c
OSD_Entry cmsMenuImuEntries[] = {
    { "-- IMU --", OME_Label, NULL, NULL, 0 },
    { "PID",       OME_Submenu, cmsMenuChange, &cmsx_menuPid, 0 },
    { "RATES",     OME_Submenu, cmsMenuChange, &cmsx_menuRates, 0 },
    { "GYRO LPF",  OME_UINT16,  NULL, &(OSD_UINT16_t){ &gyroConfigMutable()->gyro_lpf1_static_hz, 0, 1000, 1 }, 0 },
    // ...
    { "BACK",      OME_Back,    NULL, NULL, 0 },
    { NULL,        OME_END,     NULL, NULL, 0 },
};
```

Notice the data binding is **direct struct-field pointers** (not a lookup table like the CLI's `settings.c`). That's faster and smaller, but it means CMS menus must be edited C source whenever a setting is added.

### Navigation

CMS is entered by an RC mode (`BOXOSD` or a stick combo: roll + yaw + throttle held). Once in:

- **Pitch up/down**: move cursor between items.
- **Roll right**: enter submenu / increment value.
- **Roll left**: decrement value.
- **Yaw**: exit submenu.
- **Throttle**: save & exit.

`cms.c::cmsHandler()` is called as `TASK_CMS` at 60 Hz. It samples sticks, updates the menu state, and re-renders the framebuffer.

### Rendering

CMS uses the same `display_t` API as OSD (the abstract framebuffer in `drivers/display.h`). Hardware-specifically, it writes characters into the framebuffer; whether that buffer ends up on a MAX7456 chip, an OLED, or proxied over MSP-displayport to DJI goggles is the `displayport_*.c` driver's job (see [[08-io-subsystems]]).

While CMS is open, the OSD elements are suppressed — the framebuffer is owned by CMS. Exiting CMS restores normal OSD rendering.

### Interaction with arming

CMS sets `ARMING_DISABLED_CMS_MENU` while open. You cannot arm with the menu visible — protects against accidental motor spin while tuning.

## Comparison cheatsheet

| Concern | MSP | CLI | CMS |
|---------|-----|-----|-----|
| Setting binding | Direct PG field write in opcode handler | `settings.c` table → offsetof → write | Direct PG field pointer in menu entry |
| Discoverability | Wireshark / Configurator | `help`, `dump`, tab-complete | Menu tree |
| Available offline | Yes (any UART) | Yes (any UART) | No — needs OSD running |
| Available in flight | Discouraged | Blocked (`ARMING_DISABLED_CLI`) | Blocked (`ARMING_DISABLED_CMS_MENU`) |
| Adding a new setting | Add MSP opcode handler in `msp.c` | Add entry to `settings.c` | Add `OSD_Entry` to a `cms_menu_*.c` |
| Where saved to flash | All three trigger the same `writeEEPROM()` | … | … |

## Configurator workflow

For completeness, this is what a typical Configurator session looks like over MSP:

```
1. Open serial port at 115200 (or 230400 for some boards)
2. → MSP_API_VERSION                           (probe)
   ← {major=1, minor=48, mspProtocolVersion=0}
3. → MSP_FC_VARIANT                           (sniff for "BTFL")
4. → MSP_BUILD_INFO, MSP_BOARD_INFO            (display in UI)
5. → MSP_FEATURE_CONFIG, MSP_MIXER_CONFIG, MSP_RX_CONFIG,
     MSP_PID_CONTROLLER, MSP_PID, MSP_RC_TUNING,
     MSP_MOTOR_CONFIG, MSP_BATTERY_CONFIG, MSP_VOLTAGE_METERS,
     MSP_CURRENT_METERS, MSP_FAILSAFE_CONFIG, MSP_RX_FAIL_CONFIG,
     MSP_RXFAIL_CONFIG, MSP_MODE_RANGES, MSP_ADJUSTMENT_RANGES,
     MSP_BOXNAMES, MSP_BOXIDS, MSP_OSD_CONFIG, MSP_LED_STRIP_CONFIG,
     MSP_BLACKBOX_CONFIG, MSP_GPS_CONFIG, MSP_BEEPER_CONFIG,
     …                                          (hydrate UI state)
6. User changes a slider in the GUI
   → MSP_SET_PID, MSP_EEPROM_WRITE              (commit)
7. User clicks "Reboot"
   → MSP_REBOOT                                  (FC restarts)
```

There are 200+ MSP opcodes; the full list is in `msp_protocol.h` and `msp_protocol_v2_betaflight.h`. For reverse-engineering details and frame format see [[MSP Protocol]].

## See also

- [[12-config-and-pg]] for the PG system everything writes to.
- [[10-osd-blackbox-telemetry]] for the OSD layer that CMS renders onto.
- [[MSP Protocol]] for wire-format reverse engineering notes.
- [[MSP Commands Reference]] for an opcode catalog.
