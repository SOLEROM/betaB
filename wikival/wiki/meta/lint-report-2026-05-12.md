---
type: meta
title: "Wiki Lint Report 2026-05-12"
created: 2026-05-12
updated: 2026-05-12
tags: [meta, lint, health-check]
---

# Wiki Lint Report — 2026-05-12

Scope: full vault at `/mnt/betab/wikival/wiki/`
DragonScale address validation: skipped (not adopted)
DragonScale semantic tiling: skipped (not adopted)

---

## Summary

- Pages scanned: 53 (22 content pages + 20 source pages + 8 section _index.md + 3 meta)
- Issues found: 62 total
  - Critical: 30
  - Warnings: 17
  - Suggestions: 15

---

## Critical (must fix)

### C1 — Dead Wikilinks: Section Indexes (8 issues)

`wiki/index.md` and several content pages link section indexes using human-readable titles, but the actual files are all named `_index.md`. A wikilink resolver matches by filename, so `[[Features Index]]` cannot resolve to `features/_index.md`.

| Broken Link | Actual File |
|-------------|-------------|
| `[[Features Index]]` | `wiki/features/_index.md` |
| `[[Concepts Index]]` | `wiki/concepts/_index.md` |
| `[[Architecture Index]]` | `wiki/architecture/_index.md` |
| `[[Configurator Index]]` | `wiki/configurator/_index.md` |
| `[[Reverse Engineering Index]]` | `wiki/reverse/_index.md` |
| `[[Entities Index]]` | `wiki/entities/_index.md` |
| `[[Thesis Index]]` | `wiki/thesis/_index.md` |
| `[[Gaps Index]]` | `wiki/gaps/_index.md` |

Suggested fix: Rename each `_index.md` to its title (e.g., `Features Index.md`), OR replace all `[[X Index]]` links with `[[_index|X Index]]` path-qualified links, OR add an alias frontmatter field if the wiki engine supports it.

---

### C2 — Dead Wikilinks: Source Pages Named by Slug, Linked by Title (11 issues)

Eleven session-1 source pages use lowercase-slug filenames, but every content page and `index.md` links them with their human-readable title. This is an inconsistency introduced between session 1 (slug naming) and session 2 (slug naming with slug wikilinks).

| Broken Link (used in pages) | Actual Filename |
|----------------------------|-----------------|
| `[[FPV Autonomous Operation with Betaflight and Raspberry Pi]]` | `fpv-autonomous-betaflight-rpi.md` |
| `[[How to Build a 7-Inch FPV Drone (constructive extract)]]` | `build-7inch-fpv-drone.md` |
| `[[Oscar Liang FC Processors]]` | `oscarliang-fc-processors.md` |
| `[[Oscar Liang Betaflight 4.4]]` | `oscarliang-betaflight-4-4.md` |
| `[[AnyLeaf Quadcopter MCU Comparison]]` | `anyleaf-quadcopter-mcu-comparison.md` |
| `[[Betaflight Manufacturer Design Guidelines]]` | `betaflight-manufacturer-design-guidelines.md` |
| `[[Betaflight Issue 197 Supported MCUs]]` | `betaflight-issue-197-supported-mcus.md` |
| `[[Betaflight Configurator Issue 3366 F7X2 Missing]]` | `betaflight-configurator-issue-3366.md` |
| `[[DeepWiki Betaflight Config MCU Families]]` | `deepwiki-betaflight-config-mcu-families.md` |
| `[[Flying Rabbit Creating BF Target]]` | `flying-rabbit-creating-bf-target.md` |
| `[[Betaflight Getting Started Hardware]]` | `betaflight-getting-started-hardware.md` |

Affected pages (partial): `wiki/entities/STM32F722.md`, `wiki/entities/STM32 MCU Family in Betaflight.md`, `wiki/concepts/Betaflight MCU Targets.md`, `wiki/concepts/Cloud Build System.md`, `wiki/thesis/Research - STM32F8x2 (F7x2) in Betaflight.md`, and others.

Suggested fix: Pick one scheme and apply it uniformly. The cleanest resolution is to rename the 11 slug-named source files to match their human-readable title (e.g., rename `oscarliang-fc-processors.md` to `Oscar Liang FC Processors.md`). Session-2 source pages already have slugs used as wikilinks — those are consistent internally. Alternatively rename all source files to slugs and update all wikilinks to slug form.

---

### C3 — Dead Wikilinks: Missing Entity and Feature Pages (11 issues)

The following wikilinks appear in content pages but no corresponding page file exists anywhere in the vault. These are not stubs — they are pure dead links.

| Missing Page | Linked From | Priority |
|--------------|-------------|----------|
| `[[Ghidra]]` | `ARM Cortex-M Firmware Build Process`, `Cortex-M Firmware Dumping`, `Cortex-M Binary Patching`, thesis | HIGH |
| `[[SVD-Loader]]` | `Vector Table`, `Loading Cortex-M Firmware in Ghidra`, `Cortex-M Binary Patching` | HIGH |
| `[[Voltage Fault Injection]]` | `Readout Protection (STM32 RDP)`, `Cortex-M Firmware Dumping` | HIGH |
| `[[Failsafe]]` | `Li-Ion vs LiPo for FPV`, `features/_index.md` | HIGH |
| `[[GPS Rescue]]` | `Long-Range 7-Inch Class`, `MSP Override Mode`, `features/_index.md` | HIGH |
| `[[DSHOT]]` | `SpeedyBee F405 V3 BLS 50A Stack`, `entities/_index.md`, `overview.md` | MED |
| `[[Betaflight Configurator]]` | `ExpressLRS`, `entities/_index.md`, `overview.md` | MED |
| `[[OSD]]` | `SmartAudio`, `Li-Ion vs LiPo for FPV`, `overview.md` | MED |
| `[[STM32F405]]` | `STM32F722` | MED |
| `[[STM32H743]]` | `STM32F722`, `STM32 MCU Family in Betaflight` | MED |
| `[[arm-none-eabi-gcc]]` | `ARM Cortex-M Firmware Build Process` | LOW |

Suggested fix: Create stub pages for each. `[[Ghidra]]` and `[[SVD-Loader]]` are entity pages; `[[Voltage Fault Injection]]` is a concept page; `[[Failsafe]]`, `[[GPS Rescue]]`, `[[DSHOT]]`, `[[OSD]]` are feature pages. The `hot.md` "Suggested Next Steps" already flags `[[Ghidra]]` and `[[SVD-Loader]]` as priority creates.

---

### C4 — Dead Wikilinks: Additional Single-Reference Missing Pages (5 issues)

| Missing Page | Linked From |
|--------------|-------------|
| `[[ALTHOLD Mode]]` | `MSP Override Mode` |
| `[[ANGLE Mode]]` | `MSP Override Mode` |
| `[[Modes Tab]]` | `SmartAudio` (also in configurator/_index.md) |
| `[[Tramp Telemetry]]` | `SmartAudio` |
| `[[BLHeli Configurator]]` | `SpeedyBee F405 V3 BLS 50A Stack` |

Suggested fix: Create stubs in the appropriate section folders, or replace with inline descriptions and remove the brackets until pages exist.

---

### C5 — Stale Index Entries: index.md points to source pages with mismatched names (11 issues)

Same root cause as C2. The `wiki/index.md` Sources section lists 11 entries using human-readable titles (e.g., `[[Oscar Liang FC Processors]]`) that do not match any filename. Covered in C2; flagged here for priority because `index.md` is the master catalog.

---

## Warnings (should fix)

### W1 — Missing `status` Frontmatter on All `_index.md` Files (8 issues)

All eight section index files (`features/_index.md`, `concepts/_index.md`, `entities/_index.md`, `reverse/_index.md`, `architecture/_index.md`, `configurator/_index.md`, `thesis/_index.md`, `gaps/_index.md`) are missing the `status:` frontmatter field required by the vault convention.

Suggested fix: Add `status: active` (or `status: developing`) to each `_index.md` frontmatter block.

---

### W2 — Missing `status` Frontmatter on 9 Source Pages

The following session-2 source pages were created without a `status:` field:

`anvil-glitching-stm32-rdp.md`, `aticleworld-stm32-build.md`, `cjacker-opensource-toolchain-stm32.md`, `feabhas-ghidra-cortex-m.md`, `giese-defcon26-cortex-m-modify.md`, `memfault-linker-scripts.md`, `svd-loader-h2lab.md`, `techmaker-stm32-re.md`, `wrongbaud-ghidra-stm32-loader.md`

The session-1 source pages (`anyleaf-quadcopter-mcu-comparison.md`, `betaflight-*.md`, `fpv-autonomous-*.md`, `oscarliang-*.md`, `build-7inch-*.md`) have `status: ingested` — use that as the model.

Suggested fix: Add `status: ingested` to each of the 9 session-2 source files.

---

### W3 — Source Naming Convention Inconsistency (11 issues)

Session-1 source pages use slug filenames (`oscarliang-fc-processors.md`) but are wikilinked by human-readable title (`[[Oscar Liang FC Processors]]`). Session-2 source pages use slug filenames AND slug wikilinks (`[[memfault-linker-scripts]]`). The two batches use incompatible conventions, making half the source citations unresolvable.

This is a structural warning distinct from C2 (which flags the dead links). The underlying issue is that a convention was never settled. Suggested fix: decide on one scheme vault-wide and document it in `CLAUDE.md` under `## Conventions`. The simplest consistent choice: rename session-1 slugs to Title Case (matching the wikilinks that already reference them); or rename session-2 slugs to Title Case and update session-2 wikilinks. Do not mix.

---

### W4 — `overview.md` Contains Multiple Dead Wikilinks (5 issues)

`wiki/overview.md` links pages that do not exist:

| Dead Link | Notes |
|-----------|-------|
| `[[Betaflight Configurator]]` | No page file exists |
| `[[STM32]]` | No page file; closest match is `STM32 MCU Family in Betaflight` |
| `[[DSHOT]]` | No page file exists |
| `[[Research Overview]]` | No page file; separate from the thesis pages |
| `[[Open Questions]]` | No page file exists |

Suggested fix: Update `overview.md` to link the pages that do exist (`[[MSP Protocol]]`, `[[STM32 MCU Family in Betaflight]]`) and stub-out or remove the others until their pages are created.

---

### W5 — `overview.md` Uses Non-Standard `type: overview` (1 issue)

The vault schema defines these types: `feature`, `concept`, `module`, `protocol`, `entity`, `thesis`, `gap`. The value `overview` is not in the schema. This page would be excluded from type-filtered queries.

Suggested fix: Change `type: overview` to `type: meta` (or add `overview` to the schema in `CLAUDE.md`).

---

### W6 — `Research - STM32 Firmware Build and Reverse Engineering` Has Only 2 Inbound Links

This thesis page — the most detailed synthesis in the vault — is referenced only from `wiki/index.md` and `wiki/hot.md`. None of the concept or reverse engineering pages it synthesizes link back to it.

Suggested fix: Add a "See also" or "Synthesis" link to `[[Research - STM32 Firmware Build and Reverse Engineering]]` in these pages: `ARM Cortex-M Firmware Build Process`, `Cortex-M Firmware Dumping`, `Loading Cortex-M Firmware in Ghidra`, `Cortex-M Binary Patching`, `Readout Protection (STM32 RDP)`, `Vector Table`.

---

### W7 — `STM32F745` Referenced but No Entity Page Exists (1 issue)

`wiki/entities/STM32F722.md` compares the F722 against `[[STM32F745]]`, and `STM32 MCU Family in Betaflight` lists F745 in its support table. Neither creates a link target.

Suggested fix: Create a stub entity page `wiki/entities/STM32F745.md` or replace the wikilinks with plain text until a page is warranted.

---

### W8 — `CRSF Protocol` Referenced but No Page Exists (1 issue)

`wiki/entities/ExpressLRS.md` links `[[CRSF Protocol]]`. The CRSF protocol is ExpressLRS's primary data path to Betaflight and appears in `hot.md` open questions. No page exists.

Suggested fix: Create `wiki/reverse/CRSF Protocol.md` as a stub protocol page. This is flagged in the gaps index as MED priority.

---

## Suggestions (worth considering)

### S1 — Frequently Mentioned Concepts Without Dedicated Pages

The following terms appear 3+ times across content pages as unbracketed text, suggesting they are significant enough to merit pages but have not been created yet:

| Concept | Approximate Unlinked Occurrences | Suggested Page |
|---------|----------------------------------|----------------|
| `GPS Rescue` | 5 (unbracketed prose in Long-Range, MSP Override, Cortex-M Binary Patching, hot.md, overview) | `wiki/features/GPS Rescue.md` |
| `Failsafe` | 6 (Companion Computer, MSP Protocol, MSP Override Mode, Li-Ion, architecture/_index prose) | `wiki/features/Failsafe.md` |
| `Ghidra` | 7 (ARM Build Process, thesis, Cortex-M Dumping, Binary Patching — prose occurrences without `[[]]`) | `wiki/entities/Ghidra.md` |
| `radare2` | 4 (Loading in Ghidra, Cortex-M Dumping, thesis x2 — never linked) | `wiki/entities/radare2.md` or inline mention |
| `OpenOCD` | 3 (Cortex-M Dumping, ARM Build Process Stage 5, thesis) | entity stub |
| `Nexmon` | 3 (Cortex-M Binary Patching x2, thesis) | entity stub or inline |

---

### S2 — Cross-Reference Gaps Between New STM32 Build/RE Pages

The autoresearch-2 session added 6 interrelated pages (`ARM Cortex-M Firmware Build Process`, `Vector Table`, `Readout Protection (STM32 RDP)`, `Cortex-M Firmware Dumping`, `Loading Cortex-M Firmware in Ghidra`, `Cortex-M Binary Patching`). These pages link forward well but lack backward links from entity pages:

- `wiki/entities/STM32F722.md` does not link `[[ARM Cortex-M Firmware Build Process]]`, `[[Cortex-M Firmware Dumping]]`, or `[[Readout Protection (STM32 RDP)]]`, even though it discusses flash, build process, and RDP.
- `wiki/entities/STM32 MCU Family in Betaflight.md` does not link `[[Vector Table]]` or `[[ARM Cortex-M Firmware Build Process]]`.

Suggested fix: Add a "Related RE / Build" section to `STM32F722.md` and `STM32 MCU Family in Betaflight.md` linking the new concept/reverse pages.

---

### S3 — `Betaflight Configurator` Entity Needed (5 inbound references)

`[[Betaflight Configurator]]` is referenced from `ExpressLRS`, `overview.md`, `entities/_index.md`, `architecture/_index.md`, and `configurator/_index.md`. No page exists. This is the primary user-facing tool in the ecosystem.

Suggested fix: Create `wiki/entities/Betaflight Configurator.md` as a minimal entity page (what it is, where to get it, key tabs, relationship to Cloud Build).

---

### S4 — `overview.md` Has Status `stub` with No Upgrade Path

`overview.md` was created 2026-05-12 with `status: stub` and has not been expanded. The page currently links 4 dead targets (see W4) and lacks the actual overview content (only a summary `[!gap]` note).

Suggested fix: Expand `overview.md` using material already in `hot.md`, fix dead links, and change status to `developing`.

---

### S5 — Sources Section in `index.md` is Inconsistent with Actual Source File Titles

`index.md` lists `[[svd-loader-h2lab]]`, `[[aticleworld-stm32-build]]`, etc. (slug form, session 2) alongside `[[Oscar Liang FC Processors]]`, `[[AnyLeaf Quadcopter MCU Comparison]]`, etc. (title form, session 1). The catalog itself is internally inconsistent.

Suggested fix: After resolving C2/W3, update the Sources section in `index.md` so every entry uses the same convention.

---

### S6 — `[[arm-none-eabi-gcc]]` Is Linked as an Entity But Has No Page

`wiki/concepts/ARM Cortex-M Firmware Build Process.md` links `[[arm-none-eabi-gcc]]` in the "Key Relationships" section. The thesis page does the same. There is no entity page. It's a tool, not a concept, so the link style matters.

Suggested fix: Either create `wiki/entities/arm-none-eabi-gcc.md` as a minimal tool entity, or rewrite the relationship as plain text ("compiled with `arm-none-eabi-gcc`") since it is a standard toolchain component not specific to BF.

---

### S7 — `[[Research Overview]]` Referenced but Intended Page is Unclear

`wiki/thesis/_index.md` and `wiki/overview.md` both link `[[Research Overview]]`. The thesis _index.md labels it "stub" in the planned-pages table, but no file exists and the purpose overlaps with the existing thesis pages.

Suggested fix: Clarify whether `Research Overview` should be a separate synthesis page or whether `overview.md` should serve that role. If the latter, update the link in `thesis/_index.md` to point to `[[overview|Betaflight Overview]]`.

---

### S8 — `wiki/gaps/` Has Only One Filed Gap Page

`gaps/_index.md` lists 10 active gap questions, but only one page exists in `wiki/gaps/` (`STM32F8 Series Does Not Exist.md`, now resolved). The gap questions in the index are not filed as individual gap pages, making them invisible to vault-wide searches.

Suggested fix: File the top-priority gaps from `gaps/_index.md` as individual pages. Start with the MSP v2 format gap and the current BF release version gap (both HIGH priority).

---

### S9 — `wiki/sources/` Has No `_index.md`

Every other section has a `_index.md`. The `sources/` folder has none, making it invisible in the section table in `index.md`.

Suggested fix: Create `wiki/sources/_index.md` listing all 20 source pages with their type, confidence, and ingest date.

---

### S10 — `SpeedyBee F405 V3 BLS 50A Stack` Mentions `[[DSHOT]]` and `[[BLHeli Configurator]]` — Both Dead

Both links in `wiki/entities/SpeedyBee F405 V3 BLS 50A Stack.md` point to non-existent pages. Given that DSHOT is the primary ESC protocol for this stack, the dead link is particularly misleading.

Suggested fix: Create stubs `wiki/features/DSHOT.md` and resolve the BLHeli Configurator reference (either as a stub entity or plain text).

---

## Appendix: Orphan Check Results

No content pages are orphaned. All 22 content pages (features, concepts, reverse, entities, thesis, gaps) have at least 2 inbound wikilinks from other non-meta pages. The lowest-linked pages are:

- `[[Research - STM32 Firmware Build and Reverse Engineering]]` — 2 inbound links (index.md + hot.md only; no content-page inbound links — see W6)
- `[[Research - STM32F8x2 (F7x2) in Betaflight]]` — 3 inbound links
- `[[ARM Cortex-M Firmware Build Process]]` — 4 inbound links

Source pages: all 20 source pages have at least 3 inbound links from content pages (including the slug-linked session-2 sources and the title-linked session-1 sources). No source page is a true orphan, though half their wikilinks are broken (C2).

---

## Appendix: Stale Seed Pages

No pages have `status: seed`. No pages have `created:` more than 30 days before 2026-05-12. Stale-seed check: nothing to report.

---

## Appendix: Pages Over 300 Lines

No content page exceeds 300 lines. The largest pages are `wiki/entities/STM32F722.md` (~142 lines), `wiki/thesis/Research - STM32F8x2 (F7x2) in Betaflight.md` (~167 lines), and `wiki/thesis/Research - STM32 Firmware Build and Reverse Engineering.md` (~111 lines). All within limits.

---

*Report generated 2026-05-12. Do not auto-fix. Review findings and apply manually.*
