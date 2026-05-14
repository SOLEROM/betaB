# betaB — Betaflight Playground

A learning/research repo for exploring [Betaflight](https://betaflight.com/), the open-source flight control firmware for STM32-based multirotors.

## Layout

```
betab/
├── betaflight/   # upstream Betaflight repo (git submodule, read-only reference)
├── wikival/      # Obsidian research vault (notes, concepts, reverse-engineering)
└── README.md
```

- **`betaflight/`** — submodule pinned to `github.com/betaflight/betaflight.git`. Treat as read-only source-of-truth for code reading. Don't modify; pull upstream to update.
- **`wikival/`** — our knowledge base. See `wikival/CLAUDE.md` for vault conventions, note types, and `/wiki`-family workflows (ingest, query, lint, save, research).

## What we're doing here

Early-stage exploration. Goals (loose):
- Understand BF architecture: scheduler, HAL, PID loop, MSP protocol, target builds.
- Build a navigable wiki of features, modules, and reverse-engineering notes.
- Play with the codebase — no firmware changes expected yet.

## Working tips for next session

- For anything wiki-related (adding notes, ingesting sources, querying), read `wikival/CLAUDE.md` first.
- For code questions, browse `betaflight/src/` — `src/main/` is the firmware core.
- Submodule init: `git submodule update --init --recursive` if `betaflight/` is empty.
