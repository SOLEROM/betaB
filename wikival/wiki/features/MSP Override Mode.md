---
type: feature
title: "MSP Override Mode"
status: developing
area: modes
since_version: ""
created: 2026-05-12
updated: 2026-05-12
tags: [feature, msp, autonomy, modes, companion-computer]
---

# MSP Override Mode

## What It Does

MSP Override Mode allows a companion computer (e.g. Raspberry Pi) connected via USB/UART to override specific RC channels by sending `MSP_SET_RAW_RC` commands over the [[MSP Protocol]]. The flight controller substitutes the injected values in place of the corresponding receiver channels for those listed in `msp_override_channels_mask`.

This is the primary mechanism for **autonomous flight** from a companion computer while keeping BF's safety systems (PID loop, angle mode, failsafe, filters) fully active.

## How to Configure

**Step 1 — Enable mode in Configurator:**
Modes tab → assign MSP OVERRIDE to a channel/switch range (or configure it to always-on via CLI).

**Step 2 — Set the channel mask:**

```
set msp_override_channels_mask = 47
save
```

**Step 3 — Enable UART/USB MSP on the port connected to the companion computer** (Ports tab → MSP toggle).

### Key Parameters

| Parameter | Default | Notes |
|-----------|---------|-------|
| `msp_override_channels_mask` | 0 | Bitmask: bit N enables override for channel N+1 |

### CLI

```
set msp_override_channels_mask = 47   # override CH1-4 + CH6
save
```

**Value 47 decoded:** `0b00101111` — channels 1 (Roll), 2 (Pitch), 3 (Throttle), 4 (Yaw), 6 (AUX2).

## How It Works

When MSP OVERRIDE is active and `MSP_SET_RAW_FC` frames arrive, Betaflight's RX subsystem (`src/main/rx/msp.c`) writes the incoming uint16 values into the RC channel array for masked channels. From that point in the processing pipeline, the scheduler treats them identically to values from a physical receiver — they flow into mode logic, PID input mixing, and arming checks.

Non-masked channels continue to reflect the physical receiver. This allows the flight controller to remain armable via the physical TX (CH5/AUX1) while the companion computer controls attitude and throttle.

> [!key-insight] Safety architecture
> Because the FC's ANGLE mode, ALTHOLD, and failsafe all operate above the RC input layer, the companion computer cannot accidentally command a fly-away if it loses comms — BF's own failsafe will trigger on RC loss and/or MSP_SET_RAW_RC timeout.

## Interactions

- Works with: [[MSP Protocol]], [[Companion Computer]], [[ALTHOLD Mode]], [[ANGLE Mode]]
- Requires: MSP-capable UART or USB port configured in Ports tab
- Conflicts with: GPS Rescue, Acro mode (companion computer must set appropriate angle commands)
- Depends on: [[MSP Protocol]] (specifically `MSP_SET_RAW_RC`)

## Gaps / Open Questions

> [!gap] Since which BF version?
> MSP Override Mode's introduction version is unknown. Needs verification against BF changelog.

> [!gap] Timeout behavior
> What happens if `MSP_SET_RAW_RC` stops arriving? Does BF revert to receiver input, trigger failsafe, or hold last values? Not confirmed.

> [!gap] Maximum update rate
> What is the maximum rate at which `MSP_SET_RAW_RC` frames can be processed? Not specified in source article.

## Sources

- [[FPV Autonomous Operation with Betaflight and Raspberry Pi]]
