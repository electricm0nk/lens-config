---
gate: constitution-gate
initiative: terminus-inference-openaiadapter
audience: base
verdict: PASS
checked_at: 2026-04-08T00:00:00Z
---

# Constitution Gate Report — terminus-inference-openaiadapter

**Gate:** constitution-gate
**Initiative:** terminus-inference-openaiadapter
**Promotion:** large → base
**Verdict:** PASS
**Checked at:** 2026-04-08T00:00:00Z

---

## Effective Constitution

Resolved from 4 levels (additive inheritance):

| Level | File | Status |
|-------|------|--------|
| Org | `constitutions/org/constitution.md` | ✅ Loaded |
| Domain | `constitutions/terminus/constitution.md` | ✅ Loaded |
| Service | `constitutions/terminus/inference/constitution.md` | ✅ Loaded |
| Repo | `constitutions/terminus/inference/openaiadapter/` | ⬜ Not present |

---

## Compliance Results

### Org Constitution (electricm0nk)

| Article | Rule | Status | Evidence |
|---------|------|--------|---------|
| Art.1: Track Declaration | `feature` track declared | ✅ PASS | `openaiadapter.yaml: track: feature` |
| Art.2: Phase Artifacts Before Gate | Required artifacts present at each phase | ✅ PASS | `prd.md`, `architecture.md`, `tech-decisions.md`, `epics.md`, `sprint-status.yaml`, `adversarial-review-report.md`, `implementation-readiness-report-2026-04-08.md`, `stakeholder-approval.md` — all committed |
| Art.3: Architecture Documentation | `architecture.md` exists and non-empty | ✅ PASS | `docs/terminus/inference/openaiadapter/architecture.md` — 415 lines |
| Art.4: No Confidential Data Exfiltration | No cloud SaaS handling confidential data without documented justification | ✅ PASS (EXCEPTED) | OpenAI external routing is explicitly scoped and covered by Art. 15 exception (`art15-exception.md`, expiry 2027-04-07, approved ToddHintzmann). Data minimisation: `workloadClass: interactive` only — batch workload data never leaves local infrastructure (FR23). |
| Art.5: Git Discipline | All work on initiative branches, PR-only to main | ✅ PASS | Branch topology confirmed: small→medium→large→base via PRs. No direct-to-main commits. |
| Art.6: Additive Governance | Child constitutions additive only | ✅ PASS | Terminus and inference service constitutions include inheritance validation records |
| Art.7: TDD Red-Green Discipline | Failing test → min impl → refactor | ⚠️ DEFERRED | Evidence required at execution phase. All 10 dev stories include RED phase tasks in their Tasks sections. Required before story `done`. |
| Art.8: BDD Acceptance Criteria — Full Tests | No stub tests; all ACs have implemented tests | ⚠️ DEFERRED | ACs defined in all stories. Implementation deferred to dev execution — required before story `done`. |
| Art.9: Security First | No secrets in source; security best practices | ✅ PASS | Architecture documents: API key delivered exclusively via ESO/Vault → `secretKeyRef`. No key in source, Helm values, config, or logs. TLS-only to OpenAI API. |
| Art.10: Repository Is Source of Truth | Decisions committed; map-and-pointer knowledge files | ✅ PASS | All planning decisions in committed docs. Rationale captured in `architecture.md` + `tech-decisions.md`. |
| Art.11: Prefer Source-Correction; Maintain Runbook | Source corrections preferred; gaps in pre-repave-manifest | ⚠️ INFORMATIONAL | Brownfield extension — adapter added to existing gateway. No pre-repave-manifest required (no workarounds applied). Runbook article applies to infrastructure/service components; this is a feature addition. |
| Art.12: Agent Entry Point Parity | AGENTS.md + CLAUDE.md + equivalents in sync | ✅ PASS | Both `AGENTS.md` and `CLAUDE.md` present at workspace root with structural parity. Preflight sync confirmed. |
| Art.13: IDE Adapter Installation | `.claude/commands/` present | ✅ PASS | Claude Code adapter present. Preflight confirmed. |
| Art.14: Automated Phase PR Acceptance | Phase PRs auto-created and auto-merged | ✅ PASS | PR #101 (businessplan), #102 (techplan), #104 (devproposal), #106 (sprintplan) — all auto-merged per workflow. |
| Art.15: AI Safety Primacy | Art. 15 exception complete and unexpired | ✅ PASS | `art15-exception.md` committed. All 7 required items present. Expiry: 2027-04-07. Approver: ToddHintzmann. |
| Art.16: Internal Infrastructure Primacy | External service deviations documented in .todo | ✅ PASS (via Art. 15) | OpenAI (external) deviation justified in Art. 15 exception as availability mitigation. No other external dependencies. |
| Art.17: .todo Folder as Workflow Discovery Log | .todo folder present if entries exist | ✅ PASS | No unreviewed .todo entries on record for this initiative. |
| Art.18: Service Delivery Includes Deployment | Deployment artifacts scoped in sprint | ✅ PASS | Story 2.1 (ESO ExternalSecret), Story 2.2 (Helm chart secretKeyRef), Story 4.1 (smoke test against deployed gateway) — deployment and acceptance coverage included in sprint. |

### Domain Constitution (terminus)

| Article | Rule | Status | Evidence |
|---------|------|--------|---------|
| Art.1: Repo Boundaries = Service Boundaries | Feature within service repo | ✅ PASS | Initiative targets `terminus-inference-gateway` (existing service repo) — no new repository created |
| Art.2: Features Do Not Imply Repositories | Feature is planning/delivery slice | ✅ PASS | `openaiadapter` is a feature initiative targeting the existing inference service |
| Art.3: New Repos Require Justification | Any new repo has explicit justification | ✅ PASS | No new repository created |
| Art.4: Internal Network Namespace | `trantor.internal` for all hostnames | ✅ PASS | Architecture references Vault at `trantor.internal`; cluster-internal services use cluster DNS |
| Art.5: Direct-to-Main for infra/platform | N/A | ✅ N/A | Initiative targets `terminus.inference` — normal branching applies |
| Art.6: Base-to-Main Sync on Every Merge | `main` kept current with `base` | ⚠️ NOTE | Applies to target repo (`terminus-inference-gateway`) — must sync `main` after this initiative's base work lands in the gateway repo |
| Art.7: Local-First AI Data Sovereignty | Confidential data stays on `trantor.internal` | ✅ PASS (EXCEPTED) | Art. 15 exception covers OpenAI routing exception — interactive only, batch explicitly prohibited (FR23, NFR-D1). Routing policy enforces `allowedClasses: [interactive]` at config layer. |
| Art.8: AI Component Promotion Gate | Route profiles, contract tests, Art. 15 current | ✅ PASS | Route profiles defined in `architecture.md`. Contract suite in Story 1.2 (Art. 2 CI gate). Art. 15 exception valid (expiry 2027-04-07). |
| Art.9: AI Gateway Contract Stability | Single stable API contract; provider logic encapsulated | ✅ PASS | Architecture confirms: `Provider` interface encapsulates OpenAI-specific types. No OpenAI wire types leak outside `internal/adapter/openai/`. Gateway response envelope unchanged. |
| Art.10: AI Telemetry and Audit Baseline | Structured telemetry per request with required fields | ✅ PASS | Story 1.4 provides full FR17 field set: `route_id`, `provider`, `model`, `duration_ms`, `input_tokens`, `output_tokens`, `outcome`, `cost_tag` — all at INFO level. |

### Service Constitution (terminus-inference)

| Article | Rule | Status | Evidence |
|---------|------|--------|---------|
| Art.1: Route Profile Completeness | All routes have complete profiles before implementation | ⚠️ DEFERRED | Route profile manifest deferred to implementation (Story 1.3 config loader extension). Required before route is promoted. Profile values defined in `architecture.md`. |
| Art.2: Gateway Contract Gate | Contract test suite must pass before promotion | ✅ PASS | Contract suite coverage for OpenAI adapter in Story 1.2. Runs in CI via existing contract test infrastructure (Art. 2 CI gate requirement). |
| Art.3: Batch Safety Gate | Batch routes must pass safety gate | ✅ N/A | No batch routes. OpenAI explicitly prohibited from batch profile (FR23). |
| Art.4: Guardrail Requirements | All routes under active guardrails | ⚠️ DEFERRED | Guardrail values defined in NFRs (NFR-P1: ≤50ms overhead, timeout via `OPENAI_TIMEOUT_SECONDS`). Config-driven per Story 1.3. Required in route profile before story `done`. |
| Art.5: Controlled Rollout and Rollback | Rollout plan with success criteria and rollback procedure | ✅ PASS | Architecture documents: conditional registration (key absent → adapter not registered), rollback by removing `openai.keySecret` from Helm values. Art. 15 rollback plan section documents 5-step rollback procedure. |
| Art.6: No Implicit Fallback on Batch Routes | N/A | ✅ N/A | No batch routes in this initiative |

---

## Deferred Items Tracking

These items were classified as DEFERRED (not hard-gate failures). They must be resolved before their respective stories can be marked `done`:

| # | Item | Resolution Gate |
|---|------|----------------|
| D1 | TDD red-green evidence in commit history | Per-story: before `done` |
| D2 | BDD test coverage for all ACs | Per-story: before `done` |
| D3 | Route profile manifest | Story 1.3: before config loader `done` |
| D4 | Guardrail config in route profiles | Story 1.3: before config loader `done` |
| D5 | `terminus-inference-gateway` main←base sync | After execution-phase base merge |

---

## Art. 15 Exception Summary

| Field | Value |
|-------|-------|
| Exception artifact | `docs/terminus/inference/openaiadapter/art15-exception.md` |
| Approver | ToddHintzmann |
| Filed | 2026-04-07 |
| Expiry | 2027-04-07 |
| Items present | 7/7 (Decision, Threat analysis, Mitigations, Expiry, Approver, Rollback, Verification) |
| Status | VALID — not expired |

---

## Overall Verdict

**PASS** — No hard gate failures. All org-level, domain-level, and service-level constitutional requirements are either satisfied or appropriately deferred with tracked resolution gates. Art. 15 exception is valid and unexpired.

The initiative `terminus-inference-openaiadapter` is cleared to advance to `base` audience (execution-ready).
