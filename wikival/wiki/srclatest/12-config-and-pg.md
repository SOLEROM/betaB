---
type: architecture
status: stable
created: 2026-05-14
updated: 2026-05-14
tags: [config, pg, parameter-groups, eeprom, settings]
source-commit: 6434dd725
---

# 12 — Configuration & Parameter Groups

Every persistent setting in Betaflight lives in a **Parameter Group** (PG). The PG system gives three properties at once:

1. **Type-safe access.** Each PG is a C struct with named fields. The compiler ensures alignment and offsetof correctness.
2. **Versioning.** Each PG has a version number bumped when its layout changes. Mismatched versions reset to defaults instead of corrupting.
3. **Per-group dirty tracking.** Only modified PGs are written back to flash, reducing wear on a chip that has finite erase cycles.

This page covers how PGs work, how the on-flash EEPROM blob is laid out, and how all three configuration interfaces (MSP / CLI / CMS) plug into it.

## Files

| File | Role |
|------|------|
| `src/main/pg/pg.h` | Macros: `PG_REGISTER`, `PG_DECLARE`, `PG_RESET_TEMPLATE`. Registry types. |
| `src/main/pg/pg.c` | Registry walk, load, store, reset, FNV-1a hash. |
| `src/main/pg/pg_ids.h` | The big enum of PG numbers (one per group). |
| `src/main/pg/*.h` (~80 files) | Per-feature config structs (e.g. `pg/gyro.h`, `pg/rx.h`, `pg/pid.h`, `pg/osd.h`, …). |
| `src/main/config/config.c`, `.h` | Top-level: load all PGs at boot, save all dirty PGs on demand, validate cross-PG constraints. |
| `src/main/config/config_eeprom.c`, `.h`, `_impl.h` | EEPROM backend (flash / SD / RAM for SITL). |
| `src/main/config/config_streamer.c`, `.h` | Buffered write to flash, aligned to word boundary. |
| `src/main/config/feature.c`, `.h` | The feature-flag bitmask (a special "PG" of its own). |
| `src/main/cli/settings.c` | The settings table mapping CLI names to (PG, offsetof). |

## Declaring a PG

There are two registration macros:

```c
// In a header — declare an externally visible accessor function
PG_DECLARE(gyroConfig_t, gyroConfig);

// In a .c file — register the PG with the system
PG_REGISTER_WITH_RESET_TEMPLATE(gyroConfig_t, gyroConfig, PG_GYRO_CONFIG, 10);

PG_RESET_TEMPLATE(gyroConfig_t, gyroConfig,
    .gyroCalibrationDuration = 125,
    .gyro_hardware_lpf       = GYRO_HARDWARE_LPF_NORMAL,
    .gyroMovementCalibrationThreshold = 48,
    .gyro_lpf1_static_hz     = 250,
    .gyro_lpf2_static_hz     = 500,
    // ... rest of defaults
);
```

After this, code anywhere can read:

```c
const gyroConfig_t *config = gyroConfig();           // const accessor (read-only)
gyroConfig_t *mutable = gyroConfigMutable();         // mutable (marks dirty)
config->gyro_lpf1_static_hz;
mutable->gyro_lpf1_static_hz = 200;                  // automatically marks PG dirty
```

The accessors are defined by the registration macros and resolved at link time to point at the in-RAM PG struct.

**Reset variants:**

- `PG_REGISTER_WITH_RESET_TEMPLATE(...)` — the reset values are an inline struct template.
- `PG_REGISTER_WITH_RESET_FN(...)` — the reset uses a function (`pgResetFn_<name>`) for complex defaults (e.g. per-board pin assignments).
- `PG_REGISTER(...)` — no reset (zeroed on first boot).
- `PG_REGISTER_ARR(...)` — for arrays of structs (per-profile data: `PG_PID_PROFILE` has 4 entries).

## The registry

Each `PG_REGISTER` macro emits a `pgRegistry_t` entry into a special linker section (`.pg_registry`). At init, `pgIterator()` walks all entries:

```c
typedef struct pgRegistry_s {
    pgn_t       pgn;          // 12-bit ID + 4-bit version, packed
    uint8_t     length;       // array length (1 for non-array)
    uint16_t    size;         // bytes per element
    uint8_t   * address;      // pointer to in-RAM struct
    uint8_t   * copy;         // pointer to a shadow used for dirty detection (optional)
    uint8_t  ** ptr;          // pointer to update after load (rare)
    union {
        void           *ptr;  // pointer to reset template
        pgResetFunc    *fn;   // pointer to reset function
    } reset;
    uint32_t  * fnv_hash;     // FNV-1a hash of last clean state
} pgRegistry_t;
```

The registry is essentially a manifest of every persistent struct in the firmware, with hooks for default values, identification, and dirty tracking.

`pg_ids.h` defines all PG numbers:

```c
typedef enum {
    PG_SYSTEM_CONFIG         = 18,
    PG_GYRO_CONFIG           = 10,
    PG_ACCELEROMETER_CONFIG  = 11,
    PG_MOTOR_CONFIG          = 6,
    PG_RX_CONFIG             = 24,
    PG_TELEMETRY_CONFIG      = 31,
    PG_OSD_CONFIG            = 53,
    PG_PID_PROFILE           = 60,
    PG_CONTROL_RATE_PROFILES = 61,
    // ...about 80 IDs total
} PG_ID;
```

IDs are stable across firmware versions; once assigned, they are not reused. New PGs append new numbers.

## Versioning

The top 4 bits of `pgn` are a version, set at registration. When loading from EEPROM:

- If stored version equals current → load bytes (truncate if shorter, pad if longer).
- If stored version differs → don't trust the stored data, reset PG to template.

This is how the firmware survives upgrade-time layout changes. Bump the version when you remove or reorder fields. **Adding fields at the end of a PG is upgrade-safe** without bumping the version — the loader will pad new bytes with zero, then the next save writes the full new struct.

The convention is to set new field defaults via the reset template; on upgrade users get the template defaults for new fields, their existing settings preserved for old ones.

## Dirty tracking via FNV-1a hash

Each PG entry has a `fnv_hash` pointer. After a successful `pgLoad()`, the FNV-1a hash of the PG bytes is stored. On `pgStore()`, the current hash is recomputed and compared. If unchanged → skip the flash write.

This means:
- Calling `gyroConfigMutable()` doesn't directly mark dirty — it gives a mutable pointer and trusts the caller to actually change something.
- Settings writes via CLI (which goes through `settings.c::settingSet()`) explicitly invalidate by setting `configIsDirty = true`.
- MSP `MSP_SET_*` handlers also set `configIsDirty`.
- The hash check is the *final* gate before flash write — even with `configIsDirty=true`, untouched PGs aren't actually rewritten.

Net effect: a `save` writes only the PGs whose contents actually changed.

## EEPROM layout

The on-flash blob is a flat sequence:

```
[ EEPROM HEADER ]
   magic
   version (CONFIG_VERSION)
   length
[ PG record 0 ]
   pgN (12-bit ID) | pgVersion (4-bit) | size (16-bit) | length (8-bit)
   data bytes...
[ PG record 1 ]
   ...
[ ... ]
[ Footer ]
   CRC over the whole blob
```

Storage backends (`config_eeprom_impl.h`):

| Backend | Used when |
|---------|-----------|
| **Flash sector** | Default. Uses one sector of the MCU's internal flash. STM32F4: usually sector 1; STM32H7: a 128 KB sector. |
| **External flash** | If `EXTERNAL_FLASH_CONFIG` is set, config lives on the SPI flash chip alongside blackbox. |
| **SD card** | Rarely used; persists config as a file. |
| **RAM only** | SITL — config doesn't persist across runs unless saved to a host file. |

`config_streamer.c` handles the awkwardness of writing to flash: erase before write, 4-/8-byte word alignment, padding. It buffers up the entire new blob in RAM, erases the target sector, then writes the buffer.

## Boot-time load flow

`fc/init.c::initPhase2()` calls `readEEPROM()`:

```c
bool readEEPROM(void) {
    suspendRxSignal();              // pause RX state machines
    bool ok = loadEEPROM();         // pgLoad() each PG in the registry from the on-flash blob
    featureInit();                  // pre-feature bitmask hookup
    validateAndFixConfig();         // cross-PG sanity (mixer mode supported? filter cutoffs sane? GPS port assigned?)
    activateConfig();               // load active PID profile, rate profile, mixer mode
    resumeRxSignal();
    return ok;
}
```

If the EEPROM is bad (CRC fail, magic mismatch, version unknown) the whole config resets to template defaults. The user sees fresh-out-of-the-box behaviour on next boot, plus a `cliBootLogEntry` warning.

### `validateAndFixConfig()`

This is **600+ lines of cross-PG sanity** in `config.c:211-589`. It enforces consistency that PG-level reset templates can't express, because they cross PG boundaries:

- Motor protocol restrictions: PWM rate limits depending on `motor_pwm_protocol` (DShot vs PWM).
- Only one RX type enabled.
- Filter cutoffs not violating Nyquist for the current PID rate.
- `pid_process_denom` compatible with motor protocol timing.
- Mixer mode supported on this build (custom mixers compiled?).
- GPS feature requires a GPS provider AND a serial port assigned.
- Battery cell-voltage ranges sane (`vbatmincellvoltage < vbatwarningcellvoltage < vbatmaxcellvoltage`).
- OSD elements compiled-in for the current target.
- Pin assignments not conflicting (resource ownership).
- Failsafe stage 2 procedure matches available sensors (GPS rescue requires GPS).

When validation finds a conflict, the offending field is clamped to a safe value. The user sees the change on next CLI `dump`.

## Save flow

`writeEEPROM()` is the canonical save:

```c
void writeEEPROM(void) {
    rxSpiStop();    // if applicable, stop SPI-RX state machine
    systemConfigMutable()->configurationState = CONFIGURATION_STATE_CONFIGURED;
    writeUnmodifiedConfigToEEPROM();    // stream all PGs (skipped if FNV hash unchanged)
}
```

Entry points that call it:

- CLI `save` → `cliSave()` → `writeEEPROM()` + reboot.
- MSP `MSP_EEPROM_WRITE` → `writeEEPROM()`.
- CMS save-and-exit → same.
- `setProfile()` etc. don't auto-save; they require an explicit save command. This is intentional — accidental changes are recoverable until you save.

`saveConfigAndNotify()` is a convenience wrapper that bumps an in-RAM "config changed" counter so async listeners (LED status, beeper) can flash a confirmation.

## Feature flags — the special "PG" everyone uses

`featureConfig()` is a regular PG (`PG_FEATURE_CONFIG`), but the field is a 32-bit bitmask. Bits map to compile-time-ish features that can be turned on/off without recompiling:

```c
typedef enum {
    FEATURE_RX_PPM             = 1 <<  0,
    FEATURE_INFLIGHT_ACC_CAL   = 1 <<  2,
    FEATURE_RX_SERIAL          = 1 <<  3,
    FEATURE_MOTOR_STOP         = 1 <<  4,
    FEATURE_SOFTSERIAL         = 1 <<  6,
    FEATURE_GPS                = 1 <<  7,
    FEATURE_RANGEFINDER        = 1 <<  9,
    FEATURE_TELEMETRY          = 1 << 10,
    FEATURE_3D                 = 1 << 12,
    FEATURE_RX_PARALLEL_PWM    = 1 << 13,
    FEATURE_RX_MSP             = 1 << 14,
    FEATURE_RSSI_ADC           = 1 << 15,
    FEATURE_LED_STRIP          = 1 << 16,
    FEATURE_DASHBOARD          = 1 << 17,
    FEATURE_OSD                = 1 << 18,
    FEATURE_CHANNEL_FORWARDING = 1 << 20,
    FEATURE_TRANSPONDER        = 1 << 21,
    FEATURE_AIRMODE            = 1 << 22,
    FEATURE_RX_SPI             = 1 << 25,
    FEATURE_ESC_SENSOR         = 1 << 27,
    FEATURE_ANTI_GRAVITY       = 1 << 28,
    FEATURE_DYNAMIC_FILTER     = 1 << 29,
} features_e;
```

A few important conditions: `FEATURE_RX_PPM`, `FEATURE_RX_SERIAL`, `FEATURE_RX_MSP`, `FEATURE_RX_SPI`, `FEATURE_RX_PARALLEL_PWM` are mutually exclusive — `validateAndFixConfig()` enforces this by clearing all but the highest-precedence bit. Likewise `FEATURE_TELEMETRY` plus a port assigned to a telemetry function.

CLI `feature [+|-]NAME` toggles bits; CLI `feature` lists current state.

## How CLI / MSP / CMS converge

```
        ┌──────────── settings.c table ─────────────┐
        │  "altitude_lpf" → PG_POSITION + offsetof  │
        │  "gyro_lpf1_static_hz" → PG_GYRO + offset │
        │  ...about 600 settings                    │
        └──────┬────────────────────────────────────┘
               │
   CLI ─── cliSet ─┘
               │
               ▼
        cli/settings.c::settingSet()
          - validate range
          - compute target address
          - write
          - set configIsDirty
               │
        MSP_SET_* opcodes (msp.c)
          - parse payload
          - write PG struct fields directly
          - set configIsDirty
               │
        CMS menu item edit
          - menu entry holds &pgConfigMutable()->field
          - direct write
          - set configIsDirty (on menu exit)
               │
               ▼
        in-RAM PG structs (gyroConfig, rxConfig, ...)
               │
        save commands (CLI save, MSP_EEPROM_WRITE, CMS save+exit)
               │
               ▼
        writeEEPROM()
          - pgStore() each PG
            - compute FNV hash
            - if unchanged: skip
            - else: serialize, append to streamer buffer
          - config_streamer flush → flash erase + write
```

Settings have **stable names** (the `settings.c` strings). Settings have **unstable offsets** (the C struct may reshape between firmware versions). That's why backup is `dump` (set NAME=VALUE) and not "copy the flash blob".

## Profiles (a sub-system of PG)

Three things use the profile pattern — multiple instances of the same PG struct selected by an active index:

| Profile | PG | Active index field | Default count |
|---------|----|----|---------|
| PID profile | `PG_PID_PROFILE` (array of 4 `pidProfile_t`) | `systemConfig()->pidProfileIndex` | 4 |
| Rate profile | `PG_CONTROL_RATE_PROFILES` (array of 6 `controlRateConfig_t`) | `systemConfig()->activeRateProfile` | 6 |
| OSD profile | `PG_OSD_CONFIG` (array of 3 element-position sets) | `osdConfig()->osdProfileIndex` | 3 |

A `PG_REGISTER_ARR_WITH_RESET_FN` registers an array; the reset function fills every slot with template defaults. CLI `profile N`, `rateprofile N`, OSD menu select active index.

The "current" profile is referenced indirectly throughout flight code:

```c
pidController(currentPidProfile, currentTimeUs);
```

`currentPidProfile` is a global pointer updated by `setProfile()` / `activateConfig()` to point at the active slot.

## Adding a new setting (quick recipe)

1. **Add a field** to the relevant PG struct in `src/main/pg/<feature>.h`. Bump the PG's version in its `PG_REGISTER_*` call if you reordered.
2. **Set the default** in the PG's `PG_RESET_TEMPLATE` or reset function.
3. **Expose to CLI**: add an entry in `src/main/cli/settings.c` with name, type, range, PG id, offsetof.
4. **Expose to MSP** (optional): add handling in the appropriate `MSP_SET_<FEATURE>` / `MSP_<FEATURE>` opcode in `src/main/msp/msp.c`.
5. **Expose to CMS** (optional): add an `OSD_Entry` in the right `src/main/cms/cms_menu_*.c`.
6. **Read it** from flight code via `<feature>Config()->newField`.

A new setting **without bumping the PG version** is forward-compatible: old configs load and pick up the template default for the new field.

For a worked example see [[13-modification-guide]].
