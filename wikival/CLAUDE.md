# Betaflight Wiki: LLM Wiki

Mode: E (Research) + B (Repository)
Purpose: Deep-dive research into Betaflight flight control software — features, internals, configurator, and reverse engineering.
Owner: vladi.solov@gmail.com
Created: 2026-05-12

## Structure

```
vault/
├── .raw/              # source dumps, docs, changelogs, forum posts, code snippets
├── wiki/
│   ├── index.md            # master catalog of all pages
│   ├── log.md              # append-only operation log (newest first)
│   ├── hot.md              # ~500-word hot cache: recent context
│   ├── overview.md         # "What is Betaflight?" executive summary
│   ├── features/           # one page per major BF feature
│   ├── concepts/           # flight control theory, signal chain, PID math
│   ├── architecture/       # code modules, scheduler, HAL, FC targets
│   ├── configurator/       # BF Configurator tabs, CLI commands, settings
│   ├── reverse/            # MSP protocol, binary formats, build system, RE notes
│   ├── entities/           # FC hardware, ESCs, key developers, vendors
│   ├── thesis/             # evolving synthesis: "how BF actually works"
│   └── gaps/               # open questions, undocumented behavior, TODOs
├── _templates/             # frontmatter templates per note type
└── CLAUDE.md               # this file
```

## Conventions

- All notes use YAML frontmatter: type, status, created, updated, tags (minimum)
- Wikilinks use [[Note Name]] format — filenames are unique, no paths needed
- .raw/ contains source documents: never modify them
- wiki/index.md is the master catalog: update on every ingest
- wiki/log.md is append-only: never edit past entries, new entries go at the TOP
- wiki/hot.md is overwritten each session — it is a cache, not a journal

## Operations

- **Ingest**: drop source in .raw/, say "ingest [filename]"
- **Query**: ask any question — Claude reads hot.md first, then index.md, then drills in
- **Lint**: say "lint the wiki" to run a health check
- **Save**: say "save this" to file the current exchange as a wiki page
- **Research**: say "research [topic]" to trigger an autonomous web research loop

## Note Types

| Type | Folder | Template |
|------|--------|----------|
| feature | wiki/features/ | _templates/feature.md |
| concept | wiki/concepts/ | _templates/concept.md |
| module | wiki/architecture/ | _templates/module.md |
| protocol | wiki/reverse/ | _templates/protocol.md |
| entity | wiki/entities/ | _templates/entity.md |
| thesis | wiki/thesis/ | (free-form synthesis) |
| gap | wiki/gaps/ | (question + context) |
