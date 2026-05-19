---
feature: terminus-watchdog-llm-engine
doc_type: retrospective
status: approved
scope: full-feature
updated_at: 2026-05-19
sprints_covered: [sprint-0, sprint-1-e5-e1, sprint-2-e4, sprint-3-e2]
prs:
  - repo: terminus.hermes
    pr: 1
    scope: E5 + E1 (77 tests)
  - repo: terminus.hermes
    pr: 2
    scope: E4 (84 tests)
  - repo: terminus.hermes
    pr: 3
    scope: E2 (95 tests)
---

# Retrospective — Watchdog LLM Engine

**Feature:** `terminus-watchdog-llm-engine`  
**Domain/Service:** terminus/watchdog  
**Date:** 2026-05-19  
**Phase at retrospective:** dev-complete  
**Total tests delivered:** 95 (across 3 PRs)

---

## Delivery Summary

| Sprint | Scope | PRs | Tests |
|--------|-------|-----|-------|
| Sprint 0 | Infrastructure prerequisites (PRE-1/2/3) | terminus.infra #127, terminus.platform #28, terminus-inference-gateway #51 | — |
| Sprint 1 | E5 Gateway Wiring + E1 LLM Analysis Foundation | terminus.hermes PR #1 | 77 |
| Sprint 2 | E4 Interactive Remediation Wiring (Discord views + idempotency) | terminus.hermes PR #2 | 84 |
| Sprint 3 | E2 Pattern Learning Feedback Loop (batch analysis + auto-confirm SA) | terminus.hermes PR #3 | 95 |

All 16 implementation stories across 4 epics delivered. All acceptance criteria verified via TDD red-green. 95 tests passing with zero regressions across all sprint boundaries.

---

## What Went Well

### Additive Extension Architecture
The architecture decision to keep all new code in a `watchdog/triage/` package and touch existing code at exactly one integration point in `loop.py` paid off significantly. Each sprint delivered independently-testable components. No sprint had to modify code owned by a previous sprint. The detection loop never regressed.

### TDD Discipline
The red-green cycle was followed strictly across all stories. Every story started with failing tests (confirmed by running pytest before any implementation), then proceeded to green. This caught at least two subtle bugs before they reached the target branch:
- **Mock substring match bug (Story 2.1):** The query string `"HumanVerdict = \"incorrect\""` contains the word `"correct"`, which caused `"correct" in query` to return `True` even for incorrect patterns. Fixed by using quoted-string matching `'"correct" in query'` in the test assertion.
- **SA registration hang (Story 2.2):** The Temporal time-skipping test server requires custom search attributes to be pre-registered via `operator_service.add_search_attributes()` before `workflow.upsert_search_attributes()` will succeed. Without this, the workflow enters an infinite retry loop and the test hangs indefinitely.

### Temporal as the Source of Truth
Using Temporal Visibility (with Elasticsearch backend) as the observation store eliminated the SQLite schema design risk identified in the PrePlan review. Every observation is a TriageWorkflow execution — no separate database, no migration surface, no TTL management. The batch analysis graduation query is a simple Visibility API filter.

### Infrastructure Prerequisites Tracked and Resolved Early
PRE-1 (gateway Art.10 observability), PRE-2 (Elasticsearch Temporal Visibility backend), and PRE-3 (triage routing profile) were treated as a Sprint 0 that had to be verified complete before Sprint 1 began. This sequencing prevented a surprise integration failure mid-sprint.

### Temporal Search Attribute `CustomStringField` vs. Custom SA
Discovering that `CustomStringField1`–`CustomStringField10` are Temporal built-ins (pre-registered) saved Sprint 3 from complexity. Initial stories used `CustomStringField` for ad-hoc searchability. Moving `HumanVerdict` to a proper custom SA (registered via operator service) was the right choice for production queryability, and the conftest fixture pattern generalizes cleanly to any future custom SAs.

---

## What Didn't Go Well

### Sprint 0 Prerequisites Had No Story IDs
This was flagged as a HIGH finding in the FinalizePlan review and remained unresolved through dev. The PRE-1/2/3 work was tracked in `sprint-status.yaml` but had no story files, no formal BDD acceptance criteria, and no blocking indicators on E5. In practice this was fine because the infrastructure work landed cleanly, but if PRE-2 had slipped, there was no signal in the lifecycle to hold E5.

**Action:** Future features with infrastructure prerequisites should author explicit Sprint 0 story stubs in `epics.md` with dependency declarations (`blocks: [5.1]`). The Lens lifecycle can only surface blockers that are expressed as story state.

### `sprint-status.yaml` Schema Not Compatible with `complete-ops.py`
The `development_status:` flat-dict format used by the BMAD sprint-planning skill is not the `stories:` list format expected by `complete-ops.py check-preconditions`. This caused `dev_evidence_incomplete` to appear in preconditions output even though all stories are demonstrably done. The divergence is a schema mismatch between BMAD tooling and the Lens completion script.

**Action:** Either `complete-ops.py` should detect and parse the `development_status:` flat-dict format (as an alias for `stories:`), or the sprint-planning skill should emit both formats. Filed as a known gap at feature completion.

### FinalizePlan M1 Carry-Forward — Auto-Confirmed Graduation Exclusion
The BusinessPlan review flagged that `HumanVerdict=auto-confirmed` workflows should be excluded from the graduation count numerator (a silent timeout is not a human confirmation of correct behavior). This was tracked as a finding through TechPlan and FinalizePlan review. Story 2.1 implementation did exclude `auto-confirmed` from the graduation query (`HumanVerdict IN ("approved","correct")`), so the code is correct. However, the finding was never formally closed in the review documents.

**Action:** The exclusion is implemented correctly. Document in runbook: "Graduation query uses `HumanVerdict IN ("approved","correct")` — auto-confirmed and expired workflows are excluded from the numerator."

### FinalizePlan M2 — Expired Workflow Discord Button Handling Not Implemented
Story 4.1/4.2 had no AC for the case where an operator clicks an Approve/Reject button on a Discord embed for a workflow that has already expired (24-hour timeout). This was flagged in FinalizePlan M2. The current implementation will raise `WorkflowNotFoundError` from Temporal in this scenario.

**Action (post-MVP carry-forward):** Add graceful handling in the Discord button callback: catch `WorkflowNotFoundError`, return a Discord interaction response "This alert has expired — no action taken." Add to backlog as a hardening story.

---

## Technical Lessons Learned

### Temporal Time-Skipping Test Server: Custom SA Pre-Registration Required
**Symptom:** Tests hang indefinitely when calling `workflow.upsert_search_attributes()` for a custom SA that was not pre-registered.  
**Root Cause:** The time-skipping test server returns a `WorkflowTaskFailure` for unregistered SA types, which the workflow SDK retries forever.  
**Fix:** Use `autouse=True` conftest fixture that patches `WorkflowEnvironment.start_time_skipping` to call `operator_service.add_search_attributes()` after each environment start.  
**Pattern:**
```python
@pytest.fixture(autouse=True)
def _register_custom_sas(monkeypatch):
    original = WorkflowEnvironment.start_time_skipping
    async def patched(*args, **kwargs):
        env = await original(*args, **kwargs)
        await env.client.operator_service.add_search_attributes(
            AddSearchAttributesRequest(search_attributes={
                "HumanVerdict": IndexedValueType.INDEXED_VALUE_TYPE_TEXT,
                ...
            })
        )
        return env
    monkeypatch.setattr(WorkflowEnvironment, "start_time_skipping", patched)
```

### Query String Assertions Must Use Quoted-String Matching
When testing Temporal Visibility query strings, use `'"approved" in query'` rather than `"approved" in query` to avoid false positives from substrings in field names like `HumanVerdict="incorrect"` containing `"correct"`.

### Temporal `SearchAttributeKey.for_text()` vs `SearchAttributeKey.for_keyword()`
`for_text()` maps to `INDEXED_VALUE_TYPE_TEXT` (full-text indexed, appropriate for free-form strings). `for_keyword()` maps to `INDEXED_VALUE_TYPE_KEYWORD` (exact-match indexed, appropriate for enum-like fields like `HumanVerdict`). For graduation query filtering, `keyword` would be the more semantically correct type, but the difference is invisible in the time-skipping test server.

---

## Carry-Forward Backlog Items

| Priority | Item | Recommended Story |
|----------|------|-------------------|
| P1 | Expired workflow Discord button handling (graceful `WorkflowNotFoundError` → user message) | hardening-1-expired-workflow-discord-error |
| P2 | `triage_run_id_map` eviction policy (memory growth for long-running watchdog instances) | hardening-2-run-id-map-eviction |
| P3 | `TRIAGE_PROMPT_VERSION` constant for prompt versioning traceability | tech-debt-1-prompt-versioning |
| P3 | ConfigMap live-reload for `NEVER_TOUCH_CONFIG` | tech-debt-2-config-live-reload |

---

## Sprint Retrospective Summaries

### Sprint 0 — Infrastructure Prerequisites
**Outcome:** All three prerequisites delivered across `terminus.infra`, `terminus.platform`, and `terminus-inference-gateway`.  
**Key lesson:** Infrastructure prerequisites need formal story stubs with dependency declarations to be visible in lifecycle tracking.

### Sprint 1 — E5 + E1 (77 tests, PR #1)
**Outcome:** Clean delivery of the gateway wiring and full LLM triage workflow including schema, client, dispatch, scrubber, never-touch, Prometheus metrics, and Temporal workflow start.  
**Key lesson:** Art.7 TDD and Art.9 credential-scrubbing enforced cleanly from the start — no retrofit needed.

### Sprint 2 — E4 (84 tests, PR #2)
**Outcome:** Discord Approve/Reject persistent views with signal idempotency.  
**Key lesson:** Temporal signal idempotency (checking if signal already received before processing) is a must-have in any multi-user Discord interaction — duplicate button clicks are expected, not exceptional.

### Sprint 3 — E2 (95 tests, PR #3)
**Outcome:** `GraduationCandidateFinder` batch analysis and `HumanVerdict` SA upsert for Visibility queryability.  
**Key lesson:** Custom SA pre-registration in the Temporal test server is non-obvious and causes silent hangs, not errors. The autouse conftest pattern should be carried forward to any future features using custom SAs.

---

## Overall Assessment

**Delivery quality: High.** All stories delivered TDD-clean with no regression across sprint boundaries. Architecture held up — the additive extension model and Temporal-as-store decisions were both validated by the implementation.

**Process quality: Good with known gaps.** The FinalizePlan review findings were well-targeted. M2 (expired workflow button handling) is a real gap that should be addressed before production cutover. The sprint-status.yaml schema mismatch is a tooling gap, not a delivery quality issue.

**Readiness for production:** The feature is functionally complete. Before production activation, resolve the P1 carry-forward item (expired workflow Discord error handling) and verify PRE-2 Elasticsearch connectivity in the production Temporal cluster.
