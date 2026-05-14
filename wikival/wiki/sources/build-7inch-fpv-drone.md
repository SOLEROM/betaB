---
type: source
title: "How to Build a 7-Inch FPV Drone (constructive extract)"
status: ingested
source_url: https://medium.com/illumination-curated/how-to-build-an-fpv-combat-drone-for-military-purposes-ce549f24efca
raw_file: .raw/articles/build-7inch-fpv-drone-2026-05-12.md
origin_framing: military / combat
extraction_mode: constructive
created: 2026-05-12
updated: 2026-05-12
tags: [source, build-guide, 7-inch, long-range, fc, esc, vtx, elrs, li-ion]
---

# How to Build a 7-Inch FPV Drone (constructive extract)

## Provenance

> [!note] Origin honesty
> The source article was framed for combat / military use. The wiki entry
> below extracts only the **generic FPV drone engineering substrate** — the
> same know-how used by racers, freestylers, search-and-rescue volunteers,
> agricultural surveyors, infrastructure inspectors, mappers, and
> cinematographers. Payload, weaponization, and target-engagement content
> from the source is **not** reproduced here or anywhere in the vault.

This makes the page useful for the wiki's actual purpose — understanding
Betaflight and the FPV hardware ecosystem around it — without preserving
the source's destructive framing.

## Summary

A walkthrough of a 7-inch long-range FPV build using an F405-class flight
controller stack, ExpressLRS 915 MHz, analog 5.8 GHz video, and a 6S2P
Li-Ion battery. From a Betaflight perspective, the build is a textbook
**long-range cruiser** configuration: low KV motor, tri-blade 7" prop, F4
FC, DSHOT600 ESC protocol, low-band ELRS link, SmartAudio VTX control,
Li-Ion energy-density pack instead of LiPo.

## Key Technical Contributions to the Wiki

- Establishes the **7" long-range class** as a distinct configuration
  pattern in Betaflight tuning (vs 5" race / freestyle).
- Documents **Li-Ion 6S2P** battery configuration, including the BF
  cell-voltage thresholds that need to change vs LiPo defaults.
- Adds [[ExpressLRS]] (915 MHz Nano) and CRSF UART configuration as a
  long-range RC link pattern.
- Introduces **SmartAudio** as the de-facto VTX control protocol over UART
  in Betaflight.
- Documents a representative F4 stack ([[SpeedyBee F405 V3 BLS 50A Stack]])
  as a baseline reference for 30×30 FC+ESC builds.

## Pages Created from This Source

- [[Long-Range 7-Inch Class]] — concept: civilian/SAR/mapping/cinema platform
- [[Li-Ion vs LiPo for FPV]] — concept: chemistry tradeoff + BF threshold
  implications
- [[SpeedyBee F405 V3 BLS 50A Stack]] — entity: 30×30 F4 reference stack
- [[ExpressLRS]] — entity: open-source long-range RC link
- [[SmartAudio]] — feature: VTX control over UART

## Constructive Use Cases for the Same Build

The exact hardware list in this source maps cleanly onto:

- **Search-and-rescue**: long endurance from Li-Ion 6S2P; 915 MHz ELRS
  penetrates forest/urban clutter; lift budget for a thermal camera.
- **Wildfire spotting**: 7" thrust margin handles thermal payload; long
  range from low-band link.
- **Agricultural scouting**: cruise efficiency over fields, multispectral
  payload, mapping flight patterns.
- **Infrastructure inspection**: bridges, towers, pipelines — long-range
  cruise + camera tilt servo.
- **Long-range cinematography**: stable 7" platform carrying GoPro-class
  payload over distance.
- **Environmental sampling**: drop-mechanism (the article's servo trick)
  can release water-quality probes, soil sensors, or biology sample bags.

## Gaps Identified

- Whether the BF target for SpeedyBee F405 V3 is `SPEEDYBEEF405V3` or a
  unified-target variant — not confirmed against BF firmware tree.
- BF version that defaults to DSHOT600 — needs confirmation.
- Default `vbat_warning_cell_voltage` for Li-Ion vs LiPo presets — does BF
  carry a chemistry-aware preset, or is it hand-tuned each time?
- ELRS 915 MHz max link rate (LoRa-based) vs 2.4 GHz CRSF rate — to put on
  [[ExpressLRS]] page.

## Skipped Content

The source includes operational instructions for weaponizing the airframe
(payload arming, target selection, deployment). None of that is captured
in the wiki — the underlying servo-control capability already exists as
standard Betaflight functionality and is documented neutrally there.
