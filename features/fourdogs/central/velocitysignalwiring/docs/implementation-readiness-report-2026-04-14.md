---
initiative: fourdogs-central-velocitysignalwiring
track: tech-change
date: 2026-04-14
audience: medium
phase: devproposal
verdict: READY
stepsCompleted: [1, 2, 3, 4, 5, 6]
---

# Implementation Readiness Report — fourdogs-central-velocitysignalwiring

**Date:** 2026-04-14
**Track:** tech-change (architecture.md is the requirements authority)
**Audience:** medium
**Verdict:** ✅ READY

---

## Step 1: Document Discovery

| Document | Status | Notes |
|---|---|---|
| `architecture.md` | ✓ Present | Tech-change source of truth — substitutes PRD |
| `epics.md` | ✓ Present | 3 epics, 6 stories, FR coverage map |
| `adversarial-review-report.md` | ✓ Present | Verdict: PASS_WITH_NOTES (AD-2 resolved) |
| `prd.md` | N/A | Tech-change track |
| `ux-design.md` | N/A | Tech-change track |

No duplicate documents. No sharded documents. No conflicts.

---

## Step 2: Requirements Analysis

Source: `architecture.md` (tech-change track — architecture is the planning authority)

### Functional Requirements

| ID | Requirement |
|---|---|
| FR1 | Add vc TEXT column to items table via migration 022 |
| FR2 | VelocityClassifier UPDATE SQL anchored to MAX(sale_date) - 60 days |
| FR3 | Items with upw ≥ 2.0 → 'fast'; ≥ 0.5 → 'medium'; otherwise → 'slow' |
| FR4 | Items with no sales in 60-day window retain vc = NULL |
| FR5 | POST /internal/velocity/refresh endpoint triggers classifier on demand |
| FR6 | Endpoint accepts optional Bearer token auth (VELOCITY_REFRESH_TOKEN env var) |
| FR7 | Daily 4am goroutine in cmd/api calls ClassifyVelocity directly |
| FR8 | emailfetcher calls POST /internal/velocity/refresh when salesRowsImported > 0 |
| FR9 | use_vendor_catalog.ts maps row.vc → ChairSku.velocity (null/missing → 'slow') |
| FR10 | sqlc regeneration after migration makes vc appear in GetItemsByVendorRow |
| FR11 | EtailPet trigger stub RequestDailySalesReport() exists in emailfetcher, fires at most once/day |

### Non-Functional Requirements

| ID | Requirement |
|---|---|
| NFR1 | Classifier response includes duration_ms and items_classified count |
| NFR2 | NULL anchor (empty sales table) does not error — returns 0 items_classified |
| NFR3 | Tests must pass: go test ./internal/handler/... and npm test order-lifecycle |
| NFR4 | Migration is reversible (022_add_vc_to_items.down.sql drops column) |
| NFR5 | velocity.go and velocity_test.go are the canonical test surface for the classifier |
| NFR6 | No new binary — everything runs in existing fourdogs-central and emailfetcher containers |

**Total: 11 FRs + 6 NFRs = 17 requirements**

---

## Step 3: Epic Coverage Validation

| Requirement | Epic | Story | Status |
|---|---|---|---|
| FR1 | Epic 1 | Story 1.1 | ✅ Covered |
| FR2 | Epic 1 | Story 1.2 | ✅ Covered |
| FR3 | Epic 1 | Story 1.2 | ✅ Covered |
| FR4 | Epic 1 | Story 1.2 | ✅ Covered |
| FR5 | Epic 1 | Story 1.3 | ✅ Covered |
| FR6 | Epic 1 | Story 1.3 | ✅ Covered |
| FR7 | Epic 1 | Story 1.3 | ✅ Covered |
| FR8 | Epic 3 | Story 3.1 | ✅ Covered |
| FR9 | Epic 2 | Story 2.1 | ✅ Covered |
| FR10 | Epic 2 | Story 2.1 | ✅ Covered |
| FR11 | Epic 3 | Story 3.2 | ✅ Covered |
| NFR1 | Epic 1 | Stories 1.2/1.3 | ✅ Covered |
| NFR2 | Epic 1 | Story 1.2 | ✅ Covered |
| NFR3 | Epic 1+2 | Stories 1.2, 2.1 | ✅ Covered |
| NFR4 | Epic 1 | Story 1.1 | ✅ Covered |
| NFR5 | Epic 1 | Story 1.2 | ✅ Covered |
| NFR6 | Epic 3 | Stories 3.1, 3.2 | ✅ Covered |

**Coverage: 17/17 requirements covered. No gaps.**

---

## Step 4: UX Alignment

**Status: N/A (tech-change track)**

No UX design document exists or is required.

**Frontend assessment (Story 2.1):**
- Single field mapping change: `velocity: 'medium'` hardcode → `row.vc` mapping in `use_vendor_catalog.ts`
- Architecture decision AD-2 (process boundary) explicitly covers the frontend data flow
- No new UI components, workflows, or user journeys introduced
- Advisory: no separate UX spec needed at this scope

**Finding:** ADVISORY (no action required)

---

## Step 5: Epic Quality Review

### Epic Structure Validation

| Epic | User Value | Not a Tech Milestone | Independent | Result |
|---|---|---|---|---|
| Epic 1: DB Foundation + Classifier API | ✓ HOT badge gets a real data source | ✓ (not "Setup DB") | ✓ standalone | PASS |
| Epic 2: Frontend Signal Wiring | ✓ HOT badge fires from real data | ✓ (not "Build API") | ✓ depends only on E1 | PASS |
| Epic 3: emailfetcher + EtailPet Stub | ✓ imports auto-trigger reclassification | ✓ (not "Write Tests") | ✓ independent of E2 | PASS |

### Dependency Validation

- Epic 1: standalone — no upstream dependencies ✅
- Epic 2: blocked on Epic 1 Story 1.1 (sqlc regen → `GetItemsByVendorRow.Vc`) — correctly ordered ✅
- Epic 3: depends only on Epic 1 Story 1.3 (HTTP endpoint) — independent of Epic 2 ✅
- No circular dependencies ✅
- No forward dependencies in stories ✅

### Story Quality Assessment

| Story | User Value | Single-Dev Sized | Given/When/Then | Result |
|---|---|---|---|---|
| 1.1: Apply Migration + Verify vc | ✓ | ✓ | ✓ | PASS |
| 1.2: Harden Classifier + Tests | ✓ | ✓ | ✓ | PASS |
| 1.3: Wire Route, Auth, Scheduler | ✓ | ✓ | ✓ | PASS |
| 2.1: Map vc → Velocity Tier | ✓ | ✓ | ✓ | PASS |
| 3.1: emailfetcher Velocity Trigger | ✓ | ✓ | ✓ | PASS |
| 3.2: EtailPet Daily Report Stub | ✓ | ✓ | ✓ | PASS |

**Party mode findings (from adversarial review entry gate) — all pre-incorporated into stories:**
- Magos Domina: `itoa` → `strconv.FormatInt` → captured in Story 1.2 AC ✓
- Artemis: `TestClassifyVelocity_EmptySalesTable` missing → captured in Story 1.2 AC ✓
- Artemis: `sales_anchor` field in response → captured in Story 1.2 AC ✓

No open findings. No violations of epic quality standards.

---

## Step 6: Final Assessment

### Overall Readiness Status

## ✅ READY

### Summary

| Check | Result |
|---|---|
| Document inventory | 2/2 required artifacts present |
| Requirements coverage | 17/17 (100%) |
| UX alignment | N/A (tech-change) — advisory noted |
| Epic quality | 3/3 epics PASS |
| Story quality | 6/6 stories PASS |
| Dependency structure | Clean — no circular or forward references |
| Party mode findings | 3 findings, all pre-incorporated |

### Pre-existing Implementation State

This initiative has partial implementation already committed to the target branch `fourdogs-central-velocitysignalwiring`:

| File | Status |
|---|---|
| `migrations/022_add_vc_to_items.up.sql` | Exists — Story 1.1 (apply to prod) |
| `migrations/022_add_vc_to_items.down.sql` | Exists — Story 1.1 (reversible) |
| `internal/handler/velocity.go` | Exists — Story 1.2 (harden + test) |
| `internal/handler/velocity_test.go` | Exists — Story 1.2 (party mode fixes) |
| `internal/config/config.go` | Exists — Story 1.3 (wired) |
| `cmd/api/main.go` | Exists — Story 1.3 (route + scheduler wired) |

Dev agents should READ existing files before implementing — do not overwrite, only amend and polish.

### Recommendations

1. **Story 1.1** — Apply migration 022 to production DB first; run `sqlc generate` before Story 1.2 or 2.1
2. **Story 1.2** — Party mode fixes (itoa, empty sales test, sales_anchor) are targeted amendments to existing files, not rewrites
3. **Story 2.1** — Single-line frontend change; frontend repo is on initiative branch already
4. **Story 3.1 + 3.2** — emailfetcher repo requires separate checkout of initiative branch

### No Critical Issues

No blockers identified. Initiative is ready for sprint planning and implementation.
