---
feature: uifix2
doc_type: finalizeplan-review
status: pass
updated_at: "2026-04-28"
---

# FinalizePlan Review — uifix2

## Gate Status: PASS

All expressplan artifacts have been validated and are ready for developer handoff.

---

## Artifact Inventory

| Artifact | Location | Frontmatter | Adversarial Review |
|---|---|---|---|
| business-plan.md | docs/business-plan.md | ✓ pass | ✓ covered |
| tech-plan.md | docs/tech-plan.md | ✓ pass | ✓ covered |
| sprint-plan.md | docs/sprint-plan.md | ✓ pass | ✓ covered |
| expressplan-adversarial-review.md | docs/expressplan-adversarial-review.md | — | N/A (is the review) |
| stories/uifix2-1-1.md | docs/stories/uifix2-1-1.md | — | Covered by ADR-1 |
| stories/uifix2-1-2.md | docs/stories/uifix2-1-2.md | — | Covered by ADR-2 |
| stories/uifix2-1-3.md | docs/stories/uifix2-1-3.md | — | Covered by ADR-3 |
| stories/uifix2-1-4.md | docs/stories/uifix2-1-4.md | — | Covered by ADR-4 |

---

## Governance Cross-Check

**Impacted services:** `fourdogs/central` only  
**Impacted repos:** `fourdogs-central-ui` (TypeScript/React)  
**No new backend changes.** No API contract changes. No DB migrations.

**Constitution check:**
- Express track is permitted by the fourdogs domain constitution ✓
- No constitution-controlled gates blocked ✓
- `velocitysignalwiring2` dependency is live in develop ✓
- No conflicting changes detected in other active features ✓

**Cross-feature dependency scan:**
- `uifix` (fourdogs/central, phase: preplan) — distinct scope, no conflict
- `velocitysignalwiring2` (fourdogs/central, merged) — this feature fixes regressions from it; declared in `depends_on` ✓

---

## ExpressPlan Adversarial Review Summary

**Verdict:** PASS-WITH-WARNINGS  
**Critical findings:** 0  
**High findings:** 0  
**Warnings:** 4 (non-blocking, all addressed at story level)

Warnings resolved in story acceptance criteria:
- W1 (empty catalog guard) → Story uifix2-1-1 AC-4: return `[]` for empty skus
- W2 (priority badge flicker) → Story uifix2-1-2: noted, developer may add union if desired
- W3 (risk_score threshold) → Story uifix2-1-4: noted as open question
- W4 (recommendation AC scope) → Story uifix2-1-4 pre-impl verification step

---

## Open Questions Status

| Question | Owner | Status |
|---|---|---|
| Does prod ingest reliably populate items.o? | Developer / ops | Open — verify during story-4 QA |
| Should risk_score threshold be configurable? | Product owner | Open — deferred to follow-on story |
| Does risk_score > 0 produce too many items in some catalogs? | Developer | Open — story-4 developer to evaluate at test time |
| Should vc field be added to GetItemsByVendor SQL? | Developer | Deferred — explicit follow-on task |

---

## Dev Handoff Readiness

All 4 stories are ready for developer pickup. No blockers.

**Pickup order (recommended):**
1. `uifix2-1-3` (tier label gate) — simplest change, 2 lines, high signal value
2. `uifix2-1-2` (priority badge source) — straightforward 1-line useMemo fix
3. `uifix2-1-1` (catalog tabs) — requires careful empty-state handling
4. `uifix2-1-4` (recommendations button) — requires pre-impl verification step

**Target branch:** `develop` in `fourdogs-central-ui`  
**PR target:** `develop`  
**No branch promotion gate required** — this is a defect fix sprint, express track.

---

## Phase Advancement

Phase: `expressplan-complete` → `dev-ready`
