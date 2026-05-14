---
type: concept
title: "Long-Range 7-Inch Class"
status: developing
domain: hardware
created: 2026-05-12
updated: 2026-05-12
tags: [concept, hardware, long-range, 7-inch, platform-class]
---

# Long-Range 7-Inch Class

## Definition

A standard configuration of FPV multirotor optimized for **endurance and
distance** rather than agility. Identified by the propeller diameter
(7 inches) and an associated set of co-evolved choices: lower KV motors,
larger stators, higher cell count (6S typical), and frequently a Li-Ion
pack instead of LiPo.

The 7" class sits between 5" race/freestyle quads (twitchy, ~3–5 min
flights) and 10"+ cinema/long-range platforms (slow, payload-carrying,
20+ min). It is the "long-range cruiser" of the FPV world.

## Why 7" Specifically

| Aspect | 5" race/freestyle | 7" long-range | 10" cinema |
|--------|-------------------|----------------|------------|
| Typical flight time | 3–5 min | 12–25 min | 25–40 min |
| Typical KV | 1700–2400 | 1100–1500 | 800–1100 |
| Typical battery | 4S/6S LiPo | 6S Li-Ion or LiPo | 6S/8S Li-Ion |
| Pitch authority | Very high | Moderate | Lower |
| Wind tolerance | Low | Good | Good |
| Disk loading | High | Medium | Low |

7" hits the sweet spot where disk loading drops enough to extract
multi-minute endurance from realistic batteries, while staying small
enough to fold and fly aggressively when needed.

## Constructive Applications

This class is the workhorse for several pro-human use cases:

- **Search and rescue** — long endurance + thermal payload over forest,
  mountain, or urban-disaster terrain.
- **Wildfire spotting and progression mapping** — sustained cruise over
  large areas.
- **Agricultural scouting** — crop health surveys, livestock tracking,
  irrigation inspection.
- **Infrastructure inspection** — bridges, transmission towers, wind
  turbines, oil/gas pipelines.
- **Long-range cinematography** — distant subjects, follow shots over
  open water, mountain b-roll.
- **Environmental sampling** — water-quality drop probes, scientific
  sample retrieval from inaccessible terrain.

## Typical Betaflight Configuration

A representative 7" long-range build (e.g. the one documented in
[[How to Build a 7-Inch FPV Drone (constructive extract)]]) configures
Betaflight as follows:

- **ESC protocol:** DSHOT600 (F4 MCU handles this comfortably).
- **Motor mixer:** QUAD X.
- **Rate profile:** ~490 (vs ~620 default) — slightly softer rotation
  rates suit the platform's higher inertia and cruise mission.
- **Filtering:** Standard RPM filter + biquad lowpass, tuned to the lower
  motor noise spectrum of larger motors at lower RPM.
- **Battery monitor:** Cell thresholds set for chemistry — see
  [[Li-Ion vs LiPo for FPV]] for Li-Ion's flatter discharge curve.
- **GPS Rescue:** Strongly recommended; the long-range mission demands
  return-to-home behavior on link loss.
- **VTX power:** Higher (400 mW–1.6 W) for range, with **Low Power on
  Disarm** to reduce ground RF clutter.

## Hardware Pattern

| Component | Typical Spec |
|-----------|---------------|
| Frame | 7" carbon X-quad, deadcat optional for camera FOV |
| Motors | 2806–2808 stator, 1100–1500 KV |
| Props | 7×3.5 or 7×4, tri-blade |
| ESC | 4-in-1 50–60A, DSHOT600 |
| FC | F405 or F722, with baro for altitude hold |
| RC link | ExpressLRS 915 MHz (LoRa) for low-band range |
| Video | Analog 5.8 GHz 1 W+, or HDZero / DJI digital |
| Battery | Li-Ion 6S2P 8000–12000 mAh, or 6S 1500 mAh LiPo |

## Key Relationships

- Co-evolves with: [[Li-Ion vs LiPo for FPV]] (long endurance only feasible
  with Li-Ion energy density)
- Enabled by: [[ExpressLRS]] long-band link
- Tuned via: [[Rates]], [[Gyro Filtering]], [[GPS Rescue]]

> [!key-insight]
> The 7" class is what happens when you ask "how do I keep an FPV drone
> in the air for 20 minutes carrying a small payload?" Every co-evolved
> choice — KV down, stator up, cells up, prop pitch up, link band down —
> follows from that single endurance constraint.

## Sources

- [[How to Build a 7-Inch FPV Drone (constructive extract)]]
