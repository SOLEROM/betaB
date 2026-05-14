---
type: entity
title: "ExpressLRS"
entity_type: firmware
status: active
url: https://www.expresslrs.org/
created: 2026-05-12
updated: 2026-05-12
tags: [entity, firmware, rc-link, elrs, crsf, lora, 915mhz, 2.4ghz]
---

# ExpressLRS

## What It Is

ExpressLRS (ELRS) is an **open-source long-range RC link** for hobby
aircraft. It runs on cheap commodity 2.4 GHz and 915 MHz transceiver
hardware (typically based on Semtech SX127x / SX1280 LoRa chips and
Espressif ESP32 / ESP8285 MCUs) and competes with proprietary links
(TBS Crossfire, FrSky R9) on range, latency, and price — usually winning
on the latter two.

## Role in Betaflight Ecosystem

- ELRS receivers expose their data over **CRSF** (Crossfire's serial
  protocol), so Betaflight talks to ELRS via the same CRSF UART setup as
  it would to a TBS Crossfire receiver.
- Standard wiring: receiver TX → FC RX, receiver RX → FC TX, on a UART
  configured for "Serial RX" with the CRSF protocol.
- Telemetry travels back over the same CRSF link: battery voltage, RSSI,
  link-quality, GPS, and BF's `MSP` over CRSF for configurator-over-radio
  workflows.
- The "Nano" receiver form factor — about a grain of rice — fits any
  Betaflight build, including ultralight whoops.

## Frequency Bands

| Band | Hardware | Range character | Use case |
|------|----------|------------------|----------|
| 2.4 GHz | SX1280 LoRa | Long range with good antennas; less obstacle penetration | Racing, freestyle, mid-range cruising |
| 915 MHz (US) / 868 MHz (EU) | SX1276/77 LoRa | Lower data rate, deeper obstacle penetration, longer range | Long-range, search-and-rescue, mountain / forest flight |
| 433 MHz (where legal) | SX1278 | Even deeper penetration, even lower rate | Specialty long-range, scientific telemetry |

## Why It's Important to Betaflight

- It's the de-facto open-source baseline link, so BF's RX/telemetry
  configuration has converged on CRSF as the canonical receiver protocol.
- It makes long-range builds (see [[Long-Range 7-Inch Class]]) practical
  on hobbyist budgets — the same link enables search-and-rescue,
  agricultural surveys, and infrastructure inspection.
- ELRS Lua scripts on EdgeTX radios allow live BF parameter changes
  through the radio over CRSF/MSP — closing the loop between radio,
  flight controller, and firmware tuning.

## Key Facts

| Fact | Value |
|------|-------|
| License | GPL (open source) |
| Project URL | expresslrs.org / github.com/ExpressLRS |
| Underlying radio | Semtech SX127x / SX1280 LoRa |
| MCU | ESP32 / ESP8285 (most receivers/TX modules) |
| Receiver protocol to FC | CRSF over UART |
| Telemetry path | Bidirectional over CRSF |
| Update rates | up to 1000 Hz (2.4 GHz) / 200 Hz (915 MHz) |
| Configurator | ELRS Configurator (separate from BF) |

## Constructive Use-Case Fit

- **Search and rescue** — 915 MHz Nano on a 7" platform: penetrates
  forest canopy, gives the operator multi-kilometer range to locate a
  signal source.
- **Agricultural drones** — long range over fields; cheap enough that
  every aircraft in a small fleet can have the same link.
- **Educational FPV** — open-source, low-cost, hackable; ideal for
  university labs and student teams.
- **Hobby long-range / cinema** — the link that made amateur
  multi-kilometer flights mainstream.

## Related

- [[Long-Range 7-Inch Class]]
- [[SpeedyBee F405 V3 BLS 50A Stack]]
- [[CRSF Protocol]] (planned)
- [[Betaflight Configurator]]

## Gaps

> [!gap] CRSF telemetry packet structure
> The exact CRSF frame format and the subset of telemetry packets BF
> emits aren't documented in the vault yet.

> [!gap] MSP over CRSF
> The mechanism by which BF Configurator can talk to a flight controller
> over a CRSF radio link is named but not yet documented.

## Sources

- [[How to Build a 7-Inch FPV Drone (constructive extract)]]
- expresslrs.org (project documentation)
