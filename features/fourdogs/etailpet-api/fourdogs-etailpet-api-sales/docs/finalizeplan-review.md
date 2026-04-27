---
type: adversarial-review
phase: finalizeplan
feature: sales
source: manual-rerun
verdict: pass-with-warnings
reviewed-by: Todd Hintzmann
date: 2026-04-27
---

# FinalizePlan Adversarial Review — fourdogs-etailpet-api-sales

## Review Scope

Full combined planning artifact set reviewed:

| Artifact | Status |
|---|---|
| `business-plan.md` | ✓ prior session |
| `prd.md` | ✓ businessplan artifact |
| `ux-design.md` | ✓ businessplan artifact |
| `tech-plan.md` | ✓ prior session |
| `architecture.md` | ✓ techplan artifact |
| `businessplan-adversarial-review.md` | ✓ pass-with-warnings (H1+H2 accepted) |
| `techplan-adversarial-review.md` | ✓ pass-with-warnings (M1+M2 resolved) |
| `epics.md` | ✓ prior session |
| `sprint-plan.md` | ✓ prior session |
| `stories/sales-0-01` — `sales-2-03` (8 files) | ✓ prior session |

---

## Overall Verdict

**PASS-WITH-WARNINGS**

No HIGH findings. Two MEDIUM findings require action before or alongside Step 3 bundle completion. Five LOW findings are noted for awareness.

---

## Findings

### M1 — Sprint 1 Start Gate Not Enforced
**Severity:** MEDIUM
**Dimension:** Cross-Artifact Logic Flaw

`sales-0-01` (spike) is required to gate Sprint 1. However, the sprint plan does not include an explicit "Sprint 1 blocked until `sales-0-01` accepted" notation on Sprint 1 stories. A developer could begin `sales-1-01` before the spike is confirmed.

**Additionally:** `sales-0-01` acceptance criteria say "confirm" the six spike items but do not document the fail path — the consequence if one or more confirmations cannot be obtained within the 3-day timebox. The accepted businessplan risk H1 says "raise eTailPet support ticket and pause feature," but this is not visible from the story or sprint plan.

**Required action:** Before final PR merge:
- Add spike fail path to `sales-0-01` acceptance criteria: "If any confirmation item fails within the 3-day timebox → pause feature, raise eTailPet support ticket, do not start Sprint 1."
- Add a gate note to sprint plan Sprint 1 section: `⛔ Sprint 1 blocked on sales-0-01 acceptance.`

**Status:** OPEN

---

### M2 — `implementation-readiness.md` Absent From Staged Docs
**Severity:** MEDIUM
**Dimension:** Coverage Gap

The FinalizePlan bundle (Step 3) requires an implementation-readiness artifact. None exists yet. Per lifecycle contract, this is produced by `bmad-check-implementation-readiness` during Step 3 bundle execution — it is not a pre-condition for this review.

**Note:** This is a planned Step 3 output. Its absence here is expected. Recording as MEDIUM only to flag that Step 3 must not skip this artifact.

**Required action:** Produce and commit `implementation-readiness.md` during Step 3.

**Status:** EXPECTED-IN-STEP-3

---

### L1 — Date Format Requirement Not Surfaced in `sales-1-01`
**Severity:** LOW
**Dimension:** Cross-Artifact Gap

`architecture.md` documents that the www domain expects `YYYY-MM-DD` date formatting (not `MM/DD/YYYY`). This is a concrete implementation requirement. `sales-1-01` (LegacyClient story) does not reference it. A developer reading only the story will miss it.

**Required action:** Add a note to `sales-1-01` acceptance criteria referencing the date format: "All dates passed to `TriggerSalesReport` must use `YYYY-MM-DD` format per architecture.md §LegacyClient."

**Status:** OPEN (minor fix to story file)

---

### L2 — `target_repos` Empty in `feature.yaml`
**Severity:** LOW
**Dimension:** Governance Gap

`feature.yaml.target_repos` is empty. All implementation stories reference `fourdogs-central` as the target codebase. The governance record does not formally link the feature to any implementation repo. This was flagged at techplan review (L5) and remains open.

**Required action:** Update `feature.yaml` to set `target_repos: [fourdogs-central]` before or during Step 3.

**Status:** OPEN

---

### L3 — Story Estimates Undefined
**Severity:** LOW
**Dimension:** Complexity/Risk

Sprint plan uses `S` and `M` story size labels. No definition of S or M in hours or days is documented. For a solo operator with partial-time availability, this creates ambiguity in sprint duration expectations.

**Required action:** Add a legend to `sprint-plan.md`: "S ≈ half-day; M ≈ 1 full day for a single developer."

**Status:** ADVISORY (low friction fix)

---

### L4 — emailfetcher Routing Rule Dependency Has No Owner
**Severity:** LOW
**Dimension:** Cross-Feature Dependency

`sales-2-03` (staging validation) checks whether the email delivery chain works end-to-end. If the email subject line doesn't match an existing emailfetcher routing rule, the test will fail — but no owner story, PR, or contact is defined for opening a routing rule change request. The PRD Non-Goals exclude "modifying emailfetcher," but the validation story implicitly depends on it.

**Required action:** Add a note to `sales-2-03`: "If email routing rule is absent → open [emailfetcher] routing rule ticket before proceeding. This is a dependency, not in scope for this feature."

**Status:** ADVISORY

---

### L5 — Kaylee Value Chain Not Traced in Governance
**Severity:** LOW
**Dimension:** Business Value Traceability

The business case states this feature "unlocks SKU-level velocity for Kaylee." But unless a follow-on feature parses, stores, and surfaces the emailed xlsx data, the business outcome never materialises. No linked follow-on feature, epic, or placeholder exists in governance.

**Required action:** After finalizeplan completes, open a governance placeholder for the "etailpet-sales-line-ingestion" feature (or similar). Not blocking — advisory only.

**Status:** ADVISORY (post-feature)

---

## Party-Mode Challenge Round

**High Chaplain Grimaldus (Scrum Master):** The hard dependency arrow from "Sprint 0 complete" to "Sprint 1 start" is absent from the sprint plan. A developer reading the plan will not see it blocked. Fix this with a gate note.
→ *Captured as M1.*

**Lord Commander Creed (PM):** The PRD's promised business outcome (Kaylee SKU velocity) is not traceable beyond this feature's scope. Unless a follow-on feature closes the loop, the feature delivers an email trigger with no downstream consumer.
→ *Captured as L5.*

**Watch-Captain Artemis (QA):** `sales-0-01` has no explicit fail path in acceptance criteria. Under pressure, a developer will declare the spike "good enough" rather than pausing and raising support. The fail path from businessplan review H1 must be surfaced at story level.
→ *Captured as M1.*

---

## Pre-Step-3 Checklist

The following minor fixes should be applied before committing the final bundle:

| Fix | File | Status |
|---|---|---|
| Add spike fail path to `sales-0-01` AC | `stories/sales-0-01.md` | ⬜ |
| Add Sprint 1 gate note | `sprint-plan.md` | ⬜ |
| Add date format note to `sales-1-01` AC | `stories/sales-1-01.md` | ⬜ |
| Register `fourdogs-central` in `feature.yaml.target_repos` | governance | ⬜ |
| Add S/M legend to sprint plan | `sprint-plan.md` | ⬜ |
| Produce `implementation-readiness.md` (Step 3 bundle) | Step 3 | ⬜ |

---

## Risk Carry-Forward from Prior Reviews

| Finding | Source | Resolution |
|---|---|---|
| H1 — No spike fallback plan | businessplan-review | **Accepted** by Todd Hintzmann 2026-04-27 |
| H2 — Fire-and-forget monitoring gap | businessplan-review | **Accepted** by Todd Hintzmann 2026-04-27 |
| M1 — Probe HTTP server absent | techplan-review | **Resolved** in architecture.md 2026-04-27 |
| M2 — Concurrent trigger safety | techplan-review | **Resolved** in architecture.md 2026-04-27 |
