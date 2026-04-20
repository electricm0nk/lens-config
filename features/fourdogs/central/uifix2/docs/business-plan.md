---
feature: uifix2
doc_type: business-plan
status: draft
goal: >
  Resolve four UX defects in the fourdogs/central dev environment that degrade
  operator trust and workflow accuracy: vendor-mismatched catalog tabs, overly
  broad priority flags, spurious recommendation badges, and a broken "Load
  Recommendations" button.
key_decisions:
  - Fix scope is develop branch only — no schema changes required
  - Defect 4 fix uses risk_score as the primary recommendation qualifier, with suggestedQty as fallback
  - No UI redesign — fixes are minimal surgical code changes
open_questions:
  - Does the prod ingest pipeline reliably populate items.o in all environments?
  - Should risk_score threshold for auto-recommendation be configurable (default 0)?
depends_on:
  - velocitysignalwiring2 (live — dos_days, risk_score, vc fields available)
blocks: []
updated_at: "2026-04-28"
---

# Business Plan — uifix2

## Executive Summary

Four operator-facing UI defects have been observed in the fourdogs/central `develop`
environment since velocitysignalwiring2 was merged. They erode trust in the ordering
worksheet and Kaylee integration. This feature delivers targeted fixes for all four
defects in a single, no-ceremony sprint.

---

## Business Context

The Four Dogs ordering workflow is used daily by retail operators to plan purchase
orders. Operators move through two screens: a **floor walk** (scan items on the floor)
and the **ordering worksheet** (finalise quantities with Kaylee AI assistance). Defects
in either screen directly increase the cognitive load of the ordering process and can
cause incorrect order quantities or vendor-specific catalog confusion.

All four defects trace to code that was not updated when adjacent features
(velocitysignalwiring2, vendor-adapter routing) were merged:

- **Brand/category tab scoping** was never vendor-aware; vendor adapter routing made
  the mismatch visible.
- **Floor walk priority flags** always derived from local state, not server state.
- **Tier labels (INCREASED/DECREASED)** were designed for Kaylee-recommendation
  contexts but fire on all non-zero quantities.
- **Load Recommendations button** relies on `suggestedQty`, which is zero when the
  ingest pipeline runs without sales history, and which is now superseded by `risk_score`
  as the primary recommendation signal.

---

## Stakeholders

| Role | Name | Interest |
|---|---|---|
| Operator (primary user) | Four Dogs store staff | Correct, fast ordering workflow |
| Product owner | Todd | Feature quality and dev velocity |
| Developer | Dev agent | Clear, safe fix scope |

---

## Success Criteria

| # | Criterion | Measure |
|---|---|---|
| 1 | Catalog tabs show only brands/categories present in the selected vendor's catalog | Manual verification: order from SE Pet shows no Ruffwear/Tractor Supply/Reedy Fork tab unless those brands are in the catalog |
| 2 | Floor walk priority flag (amber row highlight) only marks items from a saved floor walk scan | Manual: manually typing a qty on the floor walk screen does not light the priority flag for that item |
| 3 | INCREASED / KAYLEE / DECREASED tier badges appear only after Kaylee recommendations are applied | Manual: opening the worksheet before pressing "Load Recommendations" shows no tier badges |
| 4 | "Load Recommendations" button applies quantities to at least one item when risk_score data is available | Manual: pressing the button in dev populates quantities for items with risk_score > 0 |

---

## Scope

### In scope

- `fourdogs-central-ui` (TypeScript/React) — three files changed:
  - `src/lib/catalogTabs.ts` — brand tab filtering
  - `src/pages/FloorWalk.tsx` — priority badge source data
  - `src/pages/OrderDetail.tsx` — tier label gating + recommendation button
- `fourdogs-central-ui` — one hook:
  - `src/hooks/use_vendor_catalog.ts` — ensure `riskScore`/`dosDays` fields are in the
    normalised `ChairSku` type (already mapped; verify completeness)

### Out of scope

- Backend schema changes (no new migrations)
- Backend API changes (handler/items.go is correct as-is)
- Kaylee AI recommendation engine changes
- Any feature additions beyond fixing the four defects

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Defect 4 fix changes recommendation semantics — items not previously recommended now get a qty applied | Medium | Medium | Use a conservative threshold (risk_score > 0); existing `suggestedQty` path unchanged |
| Tab-filtering change hides tabs that briefly have no items due to load state | Low | Low | Only filter after catalog data is loaded (not during loading) |
| Priority flag change breaks expected floor walk UX flow | Low | Low | Preserve scan → server-save → priority flow; new scans do not get priority until saved |
| Fix for Defect 3 removes useful signal for operators who haven't loaded Kaylee | Low | Low | Labels are removed only when kayleeQty is undefined; operator workflow calls for "Load Recommendations" first |

---

## Dependencies

- **velocitysignalwiring2** (merged, live): `dos_days`, `risk_score`, `vc` fields available
  in API response. Defect 4 fix depends on `riskScore` being populated in `ChairSku`.
- No new library or backend dependency required.

---

## Open Questions

1. Does the prod ingest pipeline reliably populate `items.o` with non-zero values?
   If yes, Defect 4 is dev-environment-only and the risk_score fallback is an
   opportunistic improvement.
2. Should the risk_score threshold for "Load Recommendations" be configurable or
   hard-coded at `> 0` (include any item with a non-zero risk score)?

---

## Timeline

Single milestone. No hard sprint boundary. All four defects are independent; each
can be developed and reviewed separately, but they ship together in one PR to `develop`.
