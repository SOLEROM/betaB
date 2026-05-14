---
type: concept
title: "Li-Ion vs LiPo for FPV"
status: developing
domain: hardware
created: 2026-05-12
updated: 2026-05-12
tags: [concept, hardware, battery, li-ion, lipo, energy-density]
---

# Li-Ion vs LiPo for FPV

## Definition

The two dominant battery chemistries used in FPV multirotors:

- **LiPo (Lithium Polymer)** — high peak current, low cell mass, short
  endurance. Standard for racing and freestyle.
- **Li-Ion (Lithium-Ion, typically 18650 / 21700 cells)** — much higher
  energy density per gram, but much lower peak current. Standard for
  long-range cruisers and payload missions.

## The Tradeoff in One Table

| Property | LiPo (race-grade) | Li-Ion (21700, e.g. P42A) |
|----------|-------------------|----------------------------|
| Energy density | ~150 Wh/kg | ~250–280 Wh/kg |
| Continuous C-rate | 60–120 C | 5–10 C |
| Peak C-rate | up to 200 C | ~15 C |
| Cell nominal | 3.7 V | 3.6 V |
| Cell min (cutoff) | 3.5 V (sag-aware) | 3.0 V (deep cutoff) |
| Discharge curve | Steep — voltage sags fast under load | Flat — voltage stays high until almost empty |
| Useful for | Racing, freestyle, peak acro | Long-range, mapping, SAR, payload |

## Why the Discharge-Curve Difference Matters in Betaflight

Betaflight's battery monitor (`vbat_warning_cell_voltage`,
`vbat_min_cell_voltage`) is calibrated against per-cell voltage. With a
**LiPo** the voltage drops steadily through the flight — at 3.5 V you have
plenty of warning. With **Li-Ion** the voltage stays flat at ~3.7 V across
most of the discharge, then drops fast at the end. A LiPo-default warning
threshold of 3.5 V will trigger almost at full charge for Li-Ion, then
the actual low-voltage cliff arrives without much margin.

### Practical thresholds

| Setting | LiPo default | Li-Ion typical |
|---------|--------------|-----------------|
| `vbat_max_cell_voltage` | 4.2 V | 4.2 V (some chemistries 4.1) |
| `vbat_warning_cell_voltage` | 3.5 V | 3.1 V |
| `vbat_min_cell_voltage` | 3.3 V | 3.0 V |

### CLI example

```
set vbat_max_cell_voltage = 420
set vbat_warning_cell_voltage = 310
set vbat_min_cell_voltage = 300
save
```

## Pack Topologies

- **6S1P** — six cells in series, one parallel branch. Compact LiPo
  geometry. Highest voltage / lowest current draw for a given power.
- **6S2P** — six in series, two in parallel. Doubles capacity and roughly
  doubles allowable continuous current. Standard Li-Ion long-range pack.
- **3S2P, 4S2P** — smaller cruisers, slow cinema platforms.

For Li-Ion the **parallel count** is the lever that recovers the C-rate
deficit: a 2P pack of 10 A cells delivers 20 A continuous, which is enough
to cruise a 7" platform even though it could never accelerate one
aggressively.

## When to Choose Which

- **LiPo** — anything where peak thrust matters more than minutes in the
  air: racing, freestyle, acro practice, indoor whoops.
- **Li-Ion** — anything where minutes matter more than peak: long-range
  cruise, search-and-rescue patterns, mapping flights, agricultural
  scouting, cinematography over distance.

## Why It Belongs in a Betaflight Wiki

Battery chemistry is not a Betaflight feature, but BF's safety behavior
(OSD warnings, audible beeper, optional auto-land via [[Failsafe]] and
[[GPS Rescue]]) only works if the configured thresholds match the
chemistry in the pack. Misconfiguration is one of the most common causes
of either nuisance landings (Li-Ion + LiPo defaults) or destroyed cells
(LiPo + Li-Ion defaults). This is the canonical "BF settings depend on
hardware physics" concept.

## Key Relationships

- Required for: [[Long-Range 7-Inch Class]]
- Configured in: BF battery monitor (`vbat_*`)
- Surfaced by: [[OSD]] cell-voltage element, [[Failsafe]]

> [!key-insight]
> Li-Ion isn't a "better LiPo." It's a different chemistry with a flat
> discharge curve, and BF's voltage-based warnings are only as good as
> the threshold-to-chemistry match.

## Sources

- [[How to Build a 7-Inch FPV Drone (constructive extract)]]
