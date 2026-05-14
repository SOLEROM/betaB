---
type: index
title: "Gaps Index"
created: 2026-05-12
updated: 2026-05-12
tags: [index, gaps]
---

# Open Questions & Gaps

Things we don't know yet, contradictions between sources, and behaviors that need investigation.

## Active Gaps

| Question | Area | Priority |
|----------|------|----------|
| What is the current stable BF release (post-4.4)? | general | HIGH |
| How does the RPM filter handle variable-RPM motors (like throttle transients)? | features/filtering | HIGH |
| What is the exact MSP v2 frame format vs MSP v1? | reverse | HIGH |
| How does the unified target system work — what is `custom_defaults`? | architecture | MED |
| What changed in BF 4.3 vs 4.4 in the PID controller? | features | MED |
| Is BLHeli_32 required for bidirectional DSHOT, or can AM32 do it? | entities | MED |
| How does iterm_relax interact with feed-forward? | concepts | MED |
| What is the exact EEPROM layout — how are pg_ sections structured? | reverse | MED |
| Does BF support DJI O3/Avata hardware? What changed for HD? | features | LOW |
| How does the BF scheduler avoid priority inversion? | architecture | LOW |

## Filing a New Gap

Use this template:

```markdown
---
type: gap
status: open
area: [features|concepts|architecture|configurator|reverse|entities]
priority: HIGH|MED|LOW
created: YYYY-MM-DD
---

# [Question]

## Context
[Why this matters / what we tried to find]

## Possible Sources
- [where to look]
```
