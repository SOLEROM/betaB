---
type: source
title: "Betaflight Configurator Issue 3366 F7X2 Missing"
source_type: github-issue
status: ingested
source_url: https://github.com/betaflight/betaflight-configurator/issues/3366
author: Configurator user (issue reporter)
date_published: 2023-03-03
confidence: medium
created: 2026-05-12
updated: 2026-05-12
tags: [source, github-issue, configurator, target, regression]
---

# Betaflight Configurator Issue 3366 F7X2 Missing

## Summary

User-filed GitHub issue reporting that Configurator **10.9.0** no
longer offered the **STM32F7X2** MCU target, which had been available
in 10.8.0. Filed 2023-03-03 by a user trying to flash BF 4.4 onto a
DIY FC built around an STM32F722.

The issue is **closed**, but the available content does not include
the resolution thread.

## Key Claims Extracted

- The target string **"STM32F7X2"** is the Configurator-side name for
  STM32F722 boards.
- The target existed in Configurator 10.8.0 and was missing in 10.9.0
  — evidence that the Configurator's MCU-target dropdown list has
  changed between releases.

## What It Contributes to the Wiki

- Confirms the **`STM32F7X2`** target string (matches what the
  unified-target build system uses; matches the Flying Rabbit blog
  post).
- Documents that **the Configurator's MCU list is not stable across
  releases** — a target can appear or disappear between versions.

Cited by:

- [[Betaflight MCU Targets]]

## Confidence

**medium**. The bare facts (target name, version regression) are clear;
the resolution and any technical reasoning behind it are not captured
in the available content.

## Provenance

Fetched 2026-05-12.
