---
type: source
title: "Oscar Liang Betaflight 4.4"
source_type: blog
status: ingested
source_url: https://oscarliang.com/betaflight-4-4/
author: Oscar Liang
confidence: high
created: 2026-05-12
updated: 2026-05-12
tags: [source, blog, oscar-liang, betaflight-4-4, cloud-build, release-notes]
---

# Oscar Liang Betaflight 4.4

## Summary

Oscar Liang's write-up of Betaflight 4.4's headline features, written
around the 4.4 release. The most important section for the wiki is the
introduction of the **Cloud Build System**, which the article explains
as a direct response to the 512 KB flash ceiling on F411 and F722
boards.

## Key Claims Extracted

- **Cloud Build was introduced in BF 4.4** as a server-side compile
  service.
- **F411 and F722 have only 512 KB of flash**; Cloud Build "can make
  Betaflight firmware smaller for these processors and prolong their
  lifespan."
- The feature is **opt-in per feature**: if you don't tick the option,
  the binary won't have the driver / capability — and the user must
  request a fresh build to add it later.
- Cloud Build is **available alongside, not in place of, traditional
  local builds**.
- The feature is forward-looking: *"in the future when Betaflight code
  continues to grow and becomes too big for some processors, it will
  come in handy."*

## What It Contributes to the Wiki

The Cloud Build motivation and mechanism. Cited from:

- [[Cloud Build System]]
- [[STM32F722]]
- [[Betaflight MCU Targets]]

## Confidence

**high** for the Cloud Build claims (Oscar quotes the Configurator UI
and tested the workflow). **medium** for any quoted developer rationale,
since Oscar paraphrases rather than directly citing the Betaflight
maintainers.

## Provenance

Fetched 2026-05-12 during the autoresearch session.
