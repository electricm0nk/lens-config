---
feature: uifix2
doc_type: adversarial-review
status: pass-with-warnings
updated_at: "2026-04-28"
---

# Adversarial Review — uifix2 ExpressPlan

**Reviewers (blind-spot challenge panel):**
- **Blind Hunter**: finds assumptions presented as facts
- **Edge Case Hunter**: finds unhandled boundary conditions
- **Acceptance Auditor**: verifies acceptance criteria are testable and sufficient

---

## Blind Hunter Findings

### B-1 (WARN) — "riskScore is already mapped" is unverified

**Location:** tech-plan ADR-4  
**Claim:** "Verify `riskScore` is mapped in `normalizeItem` (`use_vendor_catalog.ts`)"  
**Issue:** The tech-plan *assumes* the field exists in `ChairSku` from velocitysignalwiring2.
The codebase review in this session did NOT confirm `riskScore` is in `normalizeItem`.
It confirmed `suggestedQty: typeof row.o === 'number'...` and `row.dos_days` / `row.risk_score`
fields exist in the DB response, but the TypeScript normalisation line was not read.
**Verdict:** Developer must verify this in story execution. If missing, add it as part of
story 4 implementation. Not a plan defect — the story correctly calls for verification.

### B-2 (WARN) — `GetItemsByVendor` not returning `vc` — plan says "pre-existing gap, not in scope"

**Location:** tech-plan API Contracts section  
**Issue:** ADR-4 mentions `vc` as a gap and defers it. But `applyKayleeRecommendations`
sorts by velocity class for recommendation ordering. If `vc` is missing, items sort
incorrectly. The plan defers this silently.
**Verdict:** Not a blocker for the defect fix. Should be called out explicitly in the
"out of scope" section of the business plan. **Add to open questions.**

### B-3 (INFO) — Defect 4 root cause framing

**Location:** business-plan root cause section  
**Claim:** "After velocitysignalwiring2, `items.o` is zero because ingest hasn't run with
sales data in dev."  
**Issue:** This framing conflates a dev data issue with a code defect. In production,
`items.o` may be non-zero and the button works. The "broken" behaviour may be dev-only.
The fix (risk_score fallback) is an improvement, not strictly a bug fix.
**Verdict:** Acceptable framing for the purpose of this sprint — the fix is additive and
correct in both environments. Note for developers: confirm whether button works in prod.

---

## Edge Case Hunter Findings

### E-1 (WARN) — Empty catalog during tab load (ADR-1)

**Condition:** `buildCatalogTabs([])` when catalog is loading or vendor has 0 items.  
**Risk:** All tabs filter out → tab bar is empty → user sees blank navigation.  
**Current plan:** "guard with loading state (tabs only rendered after catalog load completes)"  
**Assessment:** Plan acknowledges this. Story must ensure the guard is implemented —
not just noted. Developer should either: (a) keep at least an 'everything-else' tab when
`occupied` is empty, or (b) show a loading skeleton.

### E-2 (WARN) — Priority badge disappears for in-progress scans (ADR-2)

**Condition:** Operator scans 10 items, auto-save fires every 5s. Between scans and the
next save, the just-scanned item has `lineItems` qty > 0 but is not yet in
`floorWalkLinesQuery.data` (server cache not yet invalidated).  
**Risk:** The priority badge flickers (present → absent → present after refetch).  
**Current plan:** "new scans do not get priority until saved"  
**Assessment:** Acceptable UX trade-off but should be noted in story so developer can
choose to union floorWalkLinesQuery.data with local new-scan IDs if desired. Not a
blocking concern — the badge is informational.

### E-3 (INFO) — Large risk_score catalog (ADR-4)

**Condition:** Vendor with 500+ items, all with `risk_score > 0`.  
**Risk:** "Load Recommendations" applies qtys to 500+ items at once → renders slowly,
potentially hangs the browser tab on a large `lineItems` state update.  
**Plan:** "accept for now; threshold slider in future"  
**Assessment:** Acceptable for a dev-environment fix sprint. Flag for future perf story.

### E-4 (INFO) — `caseSize || 1` when riskScore applies but suggestedQty is 0 (ADR-4)

**Condition:** Item has `riskScore = 50`, `caseSize = 12`, `suggestedQty = undefined`.  
**Result:** Applies qty = 12 (one full case).  
**Assessment:** Conservative and correct. One case is a reasonable minimum order.
No action required.

---

## Acceptance Auditor Findings

### A-1 (PASS) — Success criteria 1 (brand tabs)

Criterion: "No Ruffwear/Tractor Supply/Reedy Fork tab when those brands absent from
vendor catalog."  
Testable: Yes — verifiable in dev by ordering from SE Pet.  
Unit test: Listed in tech-plan.  
**Verdict: PASS**

### A-2 (PASS) — Success criteria 2 (priority flag)

Criterion: "Manually typing qty on floor walk screen does not light priority badge."  
Testable: Yes — manual verification.  
**Verdict: PASS**

### A-3 (PASS) — Success criteria 3 (tier labels)

Criterion: "No tier badges before Load Recommendations is pressed."  
Testable: Yes — open worksheet fresh, verify no INCREASED/DECREASED badges.  
**Verdict: PASS**

### A-4 (WARN) — Success criteria 4 (recommendation button)

Criterion: "Button applies quantities when risk_score data is available."  
**Issue:** "When risk_score data is available" is vague. Does dev actually have
`risk_score > 0` for items? The risk-refresh endpoint (`POST /internal/velocity-risk/refresh`)
must have been called after velocitysignalwiring2 deployed, and items must have non-zero
risk scores. If dev items all have `risk_score = NULL`, the button still does nothing.  
**Mitigation:** Story 4 developer should verify `risk_score` data state in dev before
declaring the fix complete. Add to story AC: "confirm `risk_score > 0` for at least one
item in dev catalog before testing."

---

## Summary

| Category | Count | Severity |
|---|---|---|
| WARN | 4 | Non-blocking, action recommended |
| INFO | 3 | Informational only |
| FAIL | 0 | — |

**Verdict: PASS-WITH-WARNINGS**

All warnings are addressed at the story level — no plan revisions required. Proceed
to sprint planning.

### Action items for story developers

1. **Story 1 (catalogTabs):** Implement loading-state guard — empty catalog should not
   render an empty tab bar. At minimum keep an 'everything-else' tab or defer rendering.
2. **Story 2 (FloorWalk priority):** Consider noting in AC that badge may flicker during
   auto-save window; decision left to developer.
3. **Story 4 (recommendations):** Verify `riskScore` mapping exists in `normalizeItem`
   and `ChairSku` type. Verify `risk_score > 0` items exist in dev DB before testing.
4. **Open question for backlog:** Add `vc` field to `GetItemsByVendor` SQL select in
   a follow-on story.
