---
initiative: terminus-inference-openaiadapter
review_type: adversarial-review
audience_gate: small_to_medium
date: '2026-04-07'
verdict: PASS_WITH_NOTES
reviewer_lead_prd: Perturabo (Architect)
reviewer_lead_arch: Lord Commander Creed (PM)
reviewers: [Perturabo, Inquisitor Greyfax, Lord Commander Creed, High Chaplain Grimaldus]
---

# Adversarial Review Report — terminus-inference-openaiadapter

**Gate:** small → medium audience promotion
**Date:** 2026-04-07
**Verdict:** PASS_WITH_NOTES

Artifacts reviewed: `prd.md`, `architecture.md`, `tech-decisions.md`, `art15-exception.md`

---

## PRD Review

**Lead:** ⚙️ Perturabo (Architect)
**Participants:** Inquisitor Greyfax (Analyst)
**Focus:** Buildable? Well-researched? UX-aligned?

### Strengths

- Requirements are cleanly partitioned across seven capability areas: adapter, routing, secrets, deployment, observability, testing, compliance.
- FR4–FR7 error mapping is precise and correct against the gateway's existing error taxonomy.
- FR17 telemetry fields match Terminus Art. 10 exactly.
- FR22 (Art. 15 exception gate) is a first-class functional requirement — not an afterthought. Artifact is now on record.
- FR23 (batch workload prohibition) explicitly documents the data sovereignty enforcement boundary.
- User journeys are coherent and traceable to specific FRs with no floating requirements.
- All referenced systems (ESO pattern, Vault delivery, routing fallback, contract suite) are grounded in the existing codebase — no assumptions were left unverified.

### Notes (P1 — Must Address in Story Authoring)

**N1 — Interface name mismatch between PRD and codebase**

The PRD's API Backend Specifics section names:
- Interface: `ProviderAdapter` (actual: `Provider`)
- Method: `Complete(ctx, CompletionRequest)` (actual: `Chat(ctx, *ChatRequest) (*ChatResponse, error)`)
- Types: `CompletionRequest` / `CompletionResponse` (actual: `ChatRequest` / `ChatResponse`)

These names do not exist in the codebase. They were assumed at PRD authoring time and corrected via source archaeology during the techplan phase. `architecture.md §Project Structure & Boundaries` is the authoritative reference.

**Action required:** DevProposal story authors must use `architecture.md` as their implementation reference, not the PRD's interface section. The PRD interface section should be treated as historical context only.

**N2 — Helm env var pattern mismatch**

PRD FR15 states `extraEnv` passthrough. Actual gateway Helm pattern is `secretKeyRef` in `env:` block. `tech-decisions.md §Authentication & Key Delivery` is authoritative.

**Action required:** Story for FR15 (Helm wiring) must reference `tech-decisions.md`, not the PRD text.

### Data Sovereignty Assessment

FR8–FR11 and FR23 correctly scope OpenAI usage to `workloadClass: interactive`. Enforcement is at the routing layer (architecturally correct — routing owns policy decisions). The adapter has no second line of defense for batch isolation. This is a known, accepted design decision documented in the PRD and in the Art. 15 exception threat analysis. No action required.

### PRD Verdict: ✅ PASS_WITH_NOTES

---

## Architecture Review

**Lead:** 🎯 Lord Commander Creed (PM)
**Participants:** Inquisitor Greyfax (Analyst), High Chaplain Grimaldus (SM)
**Focus:** Meets spec? Practical? Sprintable?

### Strengths

- All 23 FRs and 11 NFRs are explicitly mapped to architectural decisions.
- Interface ground truth is confirmed from source — no assumed names in the architecture.
- Implementation sequence (8 steps) is explicit and independently executable.
- Binding rules (error enum, telemetry fields, constructor signature, mapper naming) are precise enough to write stories directly from.
- Test strategy (`httptest.NewServer`) is concrete and avoids mock proliferation.
- Story decomposition is clean: 7–8 independently startable stories, dependency chain is linear and clear.

### Notes (P1 — Resolve Before Sprint Planning)

**N3 — Routing config schema unconfirmed**

Architecture.md identifies this as a pre-check: "confirm routing-config field names before writing FR8–FR11 stories." The provider-routing service spec is not fully visible to story authors yet. Stories for FR8 (routing config OpenAI entry) and FR9–FR11 (routing engine behavior) need to specify exact YAML field names.

**Action required:** During devproposal, confirm the provider-routing routing config YAML schema (field names, types) before writing FR8–FR11 stories. Check `terminus-inference-provider-routing` deployed config in `TargetProjects/terminus/inference/terminus-inference-provider-routing/`.

### Sprintability Assessment

| Story | Independently Startable | Dependency |
|-------|------------------------|------------|
| Adapter implementation (`adapter.go`) | ✅ Yes | None |
| Contract test suite registration | ✅ Yes | Adapter implementation |
| ESO ExternalSecret manifest | ✅ Yes | None (parallel with adapter) |
| Helm chart `secretKeyRef` wiring | ✅ Yes | None (parallel with ESO) |
| `main.go` conditional registration | ✅ Yes | Adapter implementation |
| Routing config entry (FR8–FR11) | ✅ Yes (after N3 resolved) | Schema confirmation |
| Smoke test + resilience test | Depends on deploy | All of the above deployed |

No story requires an unfinished story to be merged before it can start, except the system test (Story 8) which requires full deployment. Sprint planning is unblocked.

### Architecture Verdict: ✅ PASS_WITH_NOTES

---

## Overall Verdict

**PASS_WITH_NOTES**

The planning artifacts are sufficient to begin devproposal. No blocking issues were found. Two P1 notes must be reflected in story authoring during devproposal:

1. **N1** — Use `architecture.md` (not PRD) for interface names in implementation stories
2. **N2** — Use `tech-decisions.md` (not PRD) for Helm env var pattern in deployment story
3. **N3** — Confirm provider-routing routing config schema before writing FR8–FR11 stories

The Art. 15 exception is on record, approved by ToddHintzmann, expiring 2027-04-07.

**Next phase:** `/devproposal` on `terminus-inference-openaiadapter-medium`
