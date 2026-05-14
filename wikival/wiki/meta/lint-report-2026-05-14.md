---
type: meta
title: "Lint Report 2026-05-14"
created: 2026-05-14
updated: 2026-05-14
tags: [meta, lint, health-check]
status: developing
---

# Lint Report — 2026-05-14

Scope: full vault at `/mnt/betab/wikival/`
Trigger: user reported they "did some mess with pages" and asked to "make order".
DragonScale address validation: skipped (not adopted; no `scripts/allocate-address.sh`).
DragonScale semantic tiling: skipped (not adopted; no `scripts/tiling-check.py`).

---

## Summary

- Pages scanned: 88 markdown files under `/mnt/betab/wikival/`.
- Vault root: cleaned (was 9 stray `.md` stubs; now only `CLAUDE.md` + `README.md`).
- Auto-fixed this run: 9 (8 moves + populate, 1 delete + index/log update).
- Issues remaining: 123 dead-wikilink target names (most pre-existing, see prior [[lint-report-2026-05-12]]).
- Needs user decision: source-page naming convention, missing-page backlog.

---

## Auto-Fixed (this run)

### F1 — 8 misplaced empty stubs moved from vault root → `wiki/reverse/`

Origin: Obsidian's default "clicking a dead `[[wikilink]]` creates an empty file" behavior dropped these at the vault root instead of the proper section folder. All 8 were referenced from `wiki/reverse/_index.md`.

| File | New location | Status |
|------|--------------|--------|
| `Blackbox Format.md` | `wiki/reverse/Blackbox Format.md` | stub (frontmatter + gap callout) |
| `Bootloader.md` | `wiki/reverse/Bootloader.md` | stub |
| `Build System RE.md` | `wiki/reverse/Build System RE.md` | stub |
| `CLI Internals.md` | `wiki/reverse/CLI Internals.md` | stub |
| `CRSF Protocol.md` | `wiki/reverse/CRSF Protocol.md` | stub |
| `EEPROM Layout.md` | `wiki/reverse/EEPROM Layout.md` | stub |
| `MSP Commands Reference.md` | `wiki/reverse/MSP Commands Reference.md` | stub |
| `OSD Font Format.md` | `wiki/reverse/OSD Font Format.md` | stub |

Effect: the 8 corresponding wikilinks in `wiki/reverse/_index.md` (and elsewhere) now resolve. `wiki/index.md` Reverse Engineering section updated to list them. Page count 67 → 75.

### F2 — Empty duplicate `Oscar Liang Betaflight 4.4.md` at vault root: deleted

A 0-byte file at the vault root with this name. The real source page exists at `wiki/sources/oscarliang-betaflight-4-4.md` (slug filename). Deleting the empty stub does **not** create new dead links — the wikilink `[[Oscar Liang Betaflight 4.4]]` was already structurally broken because the source file uses slug naming (see N1 below). The empty stub was masking, not fixing, the underlying issue.

---

## Critical (must fix)

### C1 — Section-index wikilinks still dead (8)

Same as prior report C1. `wiki/index.md` and others link `[[Features Index]]`, `[[Concepts Index]]`, etc., but the actual files are all `_index.md`.

Files: `_index.md` in `features/`, `concepts/`, `architecture/`, `configurator/`, `reverse/`, `entities/`, `thesis/`, `gaps/`.

Fix options:
1. Rename each `_index.md` to its title (e.g., `Features Index.md`).
2. Add `aliases:` frontmatter to each `_index.md` (Obsidian-native; `aliases: ["Features Index"]`).
3. Replace links with path-qualified: `[[features/_index|Features Index]]`.

Option 2 is the least-invasive — preserves the `_index.md` convention and makes the wikilinks resolve.

### C2 — Source slug-vs-title naming mismatch (~9 source files)

Same as prior report C2. Session-1 sources use slug filenames but every page wikilinks them by Title Case. Effect: every citation in concept/entity/thesis pages is dead. Worst offender: `[[Betaflight Getting Started Hardware]]` (18 refs).

Slug filenames whose Title Case wikilinks are dead:
- `oscarliang-betaflight-4-4.md` ← `[[Oscar Liang Betaflight 4.4]]` (8 refs)
- `oscarliang-fc-processors.md` ← `[[Oscar Liang FC Processors]]` (6 refs)
- `anyleaf-quadcopter-mcu-comparison.md` ← `[[AnyLeaf Quadcopter MCU Comparison]]` (6 refs)
- `betaflight-manufacturer-design-guidelines.md` ← `[[Betaflight Manufacturer Design Guidelines]]` (10 refs)
- `betaflight-issue-197-supported-mcus.md` ← `[[Betaflight Issue 197 Supported MCUs]]` (7 refs)
- `betaflight-configurator-issue-3366.md` ← `[[Betaflight Configurator Issue 3366 F7X2 Missing]]` (5 refs)
- `deepwiki-betaflight-config-mcu-families.md` ← `[[DeepWiki Betaflight Config MCU Families]]` (7 refs)
- `flying-rabbit-creating-bf-target.md` ← `[[Flying Rabbit Creating BF Target]]` (7 refs)
- `betaflight-getting-started-hardware.md` ← `[[Betaflight Getting Started Hardware]]` (18 refs)
- `fpv-autonomous-betaflight-rpi.md` ← `[[FPV Autonomous Operation with Betaflight and Raspberry Pi]]` (7 refs)
- `build-7inch-fpv-drone.md` ← `[[How to Build a 7-Inch FPV Drone (constructive extract)]]` (8 refs)

Recommendation: pick **one** scheme and apply uniformly. Cleanest path: add `aliases:` to each session-1 source's frontmatter listing the Title Case form. Obsidian resolves wikilinks against aliases. Zero file renames, ~10 frontmatter edits.

### C3 — Wikilinks broken across line wraps (8 distinct, ~9 occurrences)

**New finding** not in prior report. Several wikilinks contain literal newlines because the surrounding paragraph soft-wrapped through them. Obsidian does not resolve wikilinks with embedded newlines.

| Broken link (as written) | File |
|--------------------------|------|
| `[[AnyLeaf\n  Quadcopter MCU Comparison]]` | `wiki/entities/STM32F722.md` |
| `[[AnyLeaf Quadcopter MCU\nComparison]]` | `wiki/thesis/Research - STM32F7x2 in Betaflight.md` |
| `[[Betaflight Manufacturer Design\n  Guidelines]]` | `wiki/sources/betaflight-getting-started-hardware.md` |
| `[[Betaflight Manufacturer Design\n> Guidelines]]` | `wiki/entities/STM32 MCU Family in Betaflight.md` |
| `[[Betaflight Manufacturer Design\nGuidelines]]` | `wiki/sources/betaflight-issue-197-supported-mcus.md` |
| `[[Flying Rabbit\n  Creating BF Target]]` | `wiki/concepts/Betaflight MCU Targets.md` |
| `[[Flying Rabbit Creating\nBF Target]]` | `wiki/thesis/Research - STM32F7x2 in Betaflight.md` |
| `[[Flying Rabbit Creating BF\nTarget]]` | `wiki/concepts/Betaflight MCU Targets.md` |
| `[[Oscar Liang\n  Betaflight 4.4]]` | `wiki/concepts/Cloud Build System.md`, `wiki/entities/STM32F722.md` |
| `[[Oscar Liang Betaflight\n4.4]]` | `wiki/concepts/Cloud Build System.md` |

Fix: collapse the embedded whitespace so the entire wikilink fits on one line. Trivial mechanical edit; not auto-applied this run pending your call.

### C4 — Missing high-traffic pages (still dead, refresh of C3 in prior report)

Most-referenced missing pages (sorted by inbound refs):

| Missing page | Refs | Suggested folder |
|--------------|------|------------------|
| `[[GPS Rescue]]` | 11 | `wiki/features/` |
| `[[Betaflight Configurator]]` | 7 | `wiki/entities/` |
| `[[DSHOT]]` | 6 | `wiki/features/` (or merge with `DSHOT Protocol`) |
| `[[Ghidra]]` | 5 | `wiki/entities/` |
| `[[SVD-Loader]]` | 4 | `wiki/entities/` |
| `[[Failsafe]]` | 4 | `wiki/features/` |
| `[[OSD]]` | 4 | `wiki/features/` (or alias to `OSD (On-Screen Display)`) |
| `[[Voltage Fault Injection]]` | 4 | `wiki/concepts/` |
| `[[arm-none-eabi-gcc]]` | 3 | `wiki/entities/` (or inline as plain text) |
| `[[STM32F745]]`, `[[STM32H743]]`, `[[STM32F405]]`, `[[STM32F411]]` | 2 each | `wiki/entities/` |

`[[OSD]]` collides with the existing `wiki/concepts/OSD (On-Screen Display).md`. Cleanest fix: add `aliases: ["OSD"]` to that page.

`[[DSHOT Protocol]]` is referenced 1x from `wiki/reverse/_index.md` and is the canonical form; `[[DSHOT]]` (6 refs) is the shorthand. Create one file, alias both names.

### C5 — `Configurator` section index references 12 missing tab pages

`wiki/configurator/_index.md` lists every BF Configurator tab as a wikilink, but no tab pages exist. The section is effectively empty. Pages referenced (all dead): `Configurator Welcome Tab`, `Ports Tab`, `Configuration Tab`, `Power Tab`, `Failsafe Tab`, `PID Tuning Tab`, `Rates Tab`, `Receiver Tab`, `Modes Tab`, `Adjustments Tab`, `Servos Tab`, `Motors Tab`, `LED Tab`, `OSD Tab`, `VTX Tab`, `Sensors Tab`, `Logging Tab`, `GPS Tab`, `CLI Reference`.

Suggestion: leave as a planned-pages list (replace `[[Tab]]` with `Tab` plain text) until a tab is actually written up. Otherwise every clickthrough creates another empty stub at the vault root — the exact mess we just cleaned.

### C6 — `Architecture` and `Concepts` section indexes reference many planned-but-missing pages

`wiki/architecture/_index.md` links 10 module/concept pages, none of which exist (`FC Core`, `PID Module`, `MSP Module`, `OSD Module`, `IMU Module`, etc.).
`wiki/concepts/_index.md` similarly links `Signal Chain`, `PID Theory`, `Filtering Theory`, `Gyro Sampling`, `Motor Mixing`, `Scheduler`, `ESC Protocols`, `RC Protocols`, `Feed Forward`, `Propwash`, `Rates`, `Blackbox Analysis`, `Target System`.

Same fix recommendation as C5: convert to plain-text "planned pages" list to prevent future stub-spam.

---

## Warnings

### W1 — `Research - STM32F8x2 (F7x2) in Betaflight` is a dead reference

Linked from `wiki/log.md` and `wiki/meta/lint-report-2026-05-12.md`. The actual page is `Research - STM32F7x2 in Betaflight.md` (no `F8x2`). Fix: edit the two referrer lines.

### W2 — `wiki/sources/` still has no `_index.md`

Carried from prior report S9. Every other section has one. Sources currently has 25 files.

### W3 — `wiki/overview.md` still has dead links (`STM32`, `DSHOT`, `Betaflight Configurator`, `Research Overview`, `Open Questions`)

Carried from prior report W4. Five of overview.md's wikilinks still don't resolve.

### W4 — `wiki/gaps/` is empty

Prior report S8. The gap questions live in `gaps/_index.md` as a list but are not filed as individual pages.

---

## Suggestions

### S1 — `[[lint-report-2026-05-14]]` linked from `wiki/log.md` resolves to this page

OK — this page (you're reading it) is the target. No fix needed.

### S2 — `[[Hot Cache]]` referenced from `wiki/log.md`

The hot-cache page is `wiki/hot.md`. Filename doesn't match wikilink. Either rename `hot.md` → `Hot Cache.md` or add `aliases: ["Hot Cache"]` to the frontmatter of `hot.md`.

### S3 — Inbox + canvases

- `wiki/inbox/` is empty.
- `wiki/canvases/main.canvas` and `wiki/canvases/map.canvas` — not scanned by this lint (canvases use JSON, not markdown). Verify they don't reference deleted pages.
- `.trash/main.canvas` exists at vault root — Obsidian's trash. Safe to ignore.

### S4 — Convention proposal for CLAUDE.md

Add to `wikival/CLAUDE.md` under "Conventions":

```
## Source page naming
- Source filenames may be slugs OR Title Case — both are valid.
- Every source page MUST list its human-readable title in `aliases:` frontmatter.
- Wikilinks should use the human-readable form: `[[Oscar Liang Betaflight 4.4]]`.
```

This codifies the alias-based fix for C2 and prevents the next batch of dead links.

---

## Appendix: Pages in vault root after cleanup

```
/mnt/betab/wikival/CLAUDE.md
/mnt/betab/wikival/README.md
```

Both intentional. No stray content files.

---

## Appendix: Auto-fix log

```
2026-05-14T16:36→16:55  lint --auto-fix=safe
  + mv "Blackbox Format.md"          wiki/reverse/Blackbox Format.md
  + mv "Bootloader.md"               wiki/reverse/Bootloader.md
  + mv "Build System RE.md"          wiki/reverse/Build System RE.md
  + mv "CLI Internals.md"            wiki/reverse/CLI Internals.md
  + mv "CRSF Protocol.md"            wiki/reverse/CRSF Protocol.md
  + mv "EEPROM Layout.md"            wiki/reverse/EEPROM Layout.md
  + mv "MSP Commands Reference.md"   wiki/reverse/MSP Commands Reference.md
  + mv "OSD Font Format.md"          wiki/reverse/OSD Font Format.md
  + rm "Oscar Liang Betaflight 4.4.md" (empty duplicate)
  + populate 8 stubs with proper frontmatter + gap callout + source anchors
  + update wiki/index.md (Reverse Engineering, page count 67→75)
  + prepend lint entry to wiki/log.md
```

---

## Suggested next actions (ranked)

1. **Decide on C2 alias strategy.** One pass of `aliases:` additions on ~10 source files fixes ~85 dead wikilinks vault-wide. Highest ROI.
2. **Fix C3 wrapped wikilinks** (~9 mechanical edits in 4 files). Low effort, immediate win.
3. **Fix C5/C6 stub-spam vector.** Convert "planned tabs / modules" wikilinks in section _index.md files to plain text. Stops Obsidian from re-creating empty stubs every time the user clicks one.
4. Create the 4 most-referenced stubs from C4: `Betaflight Configurator`, `GPS Rescue`, `Failsafe`, `Ghidra`. ~10 minutes each.
5. Add `aliases:` to `hot.md` (S2) and `OSD (On-Screen Display).md` (C4 aliasing).

---

*Report generated 2026-05-14. Auto-fixes applied are listed under "Auto-Fixed". All other findings are informational — no destructive action taken without further user direction.*
