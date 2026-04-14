# Product Brief: fourdogs-central-centralui50

## Executive Summary

The currently deployed Central UI is visually degraded and functionally incomplete versus the intended prototype direction. Users report enterprise-like fallback styling, missing theme controls, missing Kaylee panel behavior, and broken order flows. This initiative restores a modern, production-grade UI aligned to the existing brand intent and returns core ordering functionality.

## Problem Statement

- Visual language has regressed from prototype examples.
- Light/dark mode behavior is missing or non-functional.
- Kaylee window/panel is missing from key ordering views.
- Ordering interactions are non-functional.
- Confidence in UI deploy correctness is low after repeated regressions.

## Goals

- Restore cohesive, modern UI styling across the authenticated experience.
- Restore theme system support (light and dark).
- Restore Kaylee panel usability in ordering workflow.
- Restore end-to-end order creation/edit/submission behavior.
- Add deployment guardrails so broken UI assets cannot silently ship.

## Non-Goals

- Full UX re-platform.
- New business features outside ordering and UI reliability.
- Major backend domain redesign.

## Success Criteria

- No placeholder/fallback-only pages in authenticated flows.
- Theme toggle works and persists correctly.
- Kaylee panel renders and responds in chair phase.
- Order workflow is fully usable (create, edit, submit).
- CI/CD checks prevent asset-regression releases.
