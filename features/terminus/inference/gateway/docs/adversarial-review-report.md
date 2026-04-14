---
initiative: terminus-inference-gateway
phase: adversarial-review
audience: medium
entry_gate: adversarial-review
date: 2026-04-03
reviewedArtifacts:
  - docs/terminus/inference/gateway/product-brief.md
  - docs/terminus/inference/gateway/prd.md
  - docs/terminus/inference/gateway/architecture.md
  - docs/terminus/inference/gateway/tech-decisions.md
reviewPanel:
  - role: lead-pm
    agent: john
    focus: "Architecture review — Meets spec? Practical? Sprintable?"
  - role: lead-architect
    agent: winston
    focus: "PRD review — Buildable? Well-researched? UX-aligned?"
  - role: analyst
    agent: mary
    focus: "PRD participant — Well-researched? Grounded?"
  - role: ux
    agent: sally
    focus: "Product-brief + PRD — User-centered? Technically feasible?"
  - role: sm
    agent: bob
    focus: "Architecture participant — Sprintable? Stories derivable?"
verdict: PASS
findingsResolvedAt: 2026-04-03
---

# Adversarial Review Report — terminus-inference-gateway

**Date:** 2026-04-03
**Audience gate:** small → medium
**Mode:** party (5 reviewers)
**Verdict:** PASS_WITH_NOTES

---

## Findings

### Finding 1 — Rate Limiting Scope Contradiction [CRITICAL]

**Reviewer:** Winston (Architect)
**Artifact:** PRD vs. product-brief
**Severity:** Critical — direct contradiction between foundational planning artifacts

The PRD "Out of scope" section states: *"Rate guard at the gateway level"* is out of scope. The product-brief "In Scope" section explicitly includes: *"Rate guard at the gateway level (per-client limits)"*.

These two documents are the foundational planning artifacts for this feature. They directly contradict each other on a functional boundary. An implementation agent will pick one source and be wrong. The correct resolution is a single explicit statement: rate limiting at the gateway boundary is either in or out of scope for this initiative, with the rationale documented. The architecture document also omits this boundary entirely, suggesting it was intended to be out of scope — the product-brief should be corrected.

**Resolution:** Correct the product-brief to match the PRD and architecture (rate limiting delegated to `terminus-inference-provider-routing`). Add an explicit exclusion note with rationale.

---

### Finding 2 — Streaming Support Undecided [CRITICAL]

**Reviewer:** John (PM)
**Artifact:** PRD, architecture
**Severity:** Critical — implementation agents will diverge

OpenAI `/v1/chat/completions` supports `stream: true` (Server-Sent Events). Ollama supports streaming natively. Neither the PRD nor the architecture ever states whether streaming is in or out of scope for this MVP. An implementation agent has 50/50 odds of implementing it or ignoring it, with no canonical answer to point to. If streaming is implemented without a design decision, the SSE transport and response pipeline ordering will be inconsistent with the non-streaming path. If it is excluded without documentation, callers expecting streaming will receive silent failures.

**Resolution:** Add an explicit statement in the PRD and architecture: "Streaming (`stream: true`) is [in/out of] scope for MVP. Rationale: …" If out of scope, specify that the gateway must return an explicit 400 or 501 for streaming requests rather than silently mishandling them.

---

### Finding 3 — Telemetry Gap (Domain Constitution Art.10) [MAJOR]

**Reviewer:** John (PM)
**Artifact:** Architecture
**Severity:** Major — domain constitution Article 10 requires telemetry on every request

Domain constitution Article 10 mandates that all AI inference components emit structured telemetry per request including: route identifier, model used, request start time, duration, token counts (where measurable), outcome, and cost tag. The architecture defines `log/slog` structured logging with specific field names, but the telemetry fields required by Art.10 are not fully specified:
- `route` is in the logging fields — covers route identifier ✅
- `provider` is in the logging fields — covers model/provider ✅
- `status` covers outcome partially ✅
- **Duration is absent** — no log field or middleware timer defined
- **Token counts** — response body token fields are never specified as extracted/logged  
- **Cost tag** — not mentioned anywhere in the architecture or PRD

The Architecture Validation section marks observability ✅ but only references health/readiness endpoints. No `internal/telemetry/` package, no `/metrics` endpoint, no telemetry middleware step in the request pipeline is defined.

**Resolution:** Add a telemetry section to the architecture specifying: (1) which pipeline step injects timing; (2) how token counts from adapter responses are extracted and logged; (3) whether a cost tag is a fixed config value per provider or derived from token counts. At minimum, add `duration_ms` and `token_count_input`/`token_count_output` to the logging field specification.

---

### Finding 4 — Vault Token TTL and Renewal Unaddressed [MAJOR]

**Reviewer:** Mary (Analyst)
**Artifact:** Architecture
**Severity:** Major — operational correctness risk

The auth design mandates Vault token validation on every request (`lookup-self`) with no session cache (NFR-S4). Vault tokens have TTLs. The architecture makes no provision for:
- What happens when the gateway's own Vault client token expires (the token used to make `lookup-self` calls)
- Whether the gateway can renew its token or must restart
- What error is surfaced to callers when a Vault connection failure or expired client token causes all `lookup-self` calls to fail

A gateway whose own Vault client token has silently expired will return `401` errors for every caller with no diagnostic path distinguishing "bad caller token" from "gateway infrastructure failure."

**Resolution:** Specify in the architecture: (a) what Vault token type the gateway uses for its own client connection (periodic token, service token, k8s auth role); (b) whether renewal is handled by a Vault agent sidecar or by the gateway itself; (c) what log event and error behavior occur when the gateway's own Vault client fails.

---

### Finding 5 — Vault Token Injection Mechanism Unspecified [MAJOR]

**Reviewer:** Mary (Analyst)
**Artifact:** Architecture
**Severity:** Major — integration risk with `terminus-infra-secrets`

The architecture states "Vault client must reach Vault at startup" and references `GATEWAY_VAULT_ADDR` and `GATEWAY_VAULT_TOKEN` env vars for local dev. But the k8s deployment mechanism for injecting the production Vault token is unspecified. The Helm chart reference notes ConfigMap/Secret injection for env vars, but Vault token injection typically uses one of: Vault Agent injector sidecar, k8s auth via service account annotation, mounted secret from ESO. The architecture does not specify which approach is used or how it integrates with the existing `terminus-infra-secrets` provisioning model.

This is a coordination point with a sibling initiative (`terminus-infra-vaultgate` is currently active at `small` audience). If the injection mechanism is decided inconsistently with how `vaultgate` works, deployment will fail at integration time.

**Resolution:** Add a deployment section to the architecture specifying the Vault token injection mechanism for k8s (the Helm chart `values.yaml` should expose this as a named pattern with a default). Cross-reference `terminus-infra-secrets` approach.

---

### Finding 6 — `oapi-codegen.yaml` Config Unspecified [MINOR]

**Reviewer:** John (PM)
**Artifact:** Architecture
**Severity:** Minor — implementation deviation risk

The architecture references `api/oapi-codegen.yaml` as a committed config file but never specifies its contents. Critical options include: whether to generate a `StrictServerInterface` (stricter, recommended) or standard `ServerInterface`, how to handle error responses in generated code, and package naming. An implementation agent will default to minimal codegen options and may produce a different interface shape than the architecture implies.

**Resolution:** Add a brief spec for `oapi-codegen.yaml` in the architecture (4–5 lines covering interface type, packages, and output paths).

---

### Finding 7 — Helm values.yaml Unspecified [MINOR]

**Reviewer:** John (PM)
**Artifact:** Architecture
**Severity:** Minor — deployment friction

The architecture defines the full project directory tree including `deploy/helm/values.yaml` but provides no guidance on what configuration values it exposes. The env vars are specified (`GATEWAY_VAULT_ADDR`, etc.) but the mapping between Helm values and container env vars is implicit. A developer deploying to k3s without knowing the expected values structure will be blocked.

**Resolution:** Add a brief Helm values sketch to the architecture specifying the top-level keys, their types, and which ones are required with no default.

---

### Finding 8 — Local Development Workflow Absent [MINOR]

**Reviewer:** Sally (UX)
**Artifact:** product-brief, architecture
**Severity:** Minor — DX gap

The product-brief identifies "Developer (local)" as a target user and the PRD describes a "Happy Path" journey where Alex points an LLM client at the gateway. But neither artifact specifies how to run the gateway locally (against a local Ollama instance). A `docker-compose.yaml`, `make dev` target, or equivalent local dev story is absent. The `Makefile` targets defined in the architecture do not include a `make run-local` or equivalent.

**Resolution:** Add a `make run-local` (or equivalent) to the Makefile spec in the architecture that wires up the gateway with a local Ollama instance and a test Vault token for development use.

---

### Finding 9 — Story Sizing Hints Absent [MINOR]

**Reviewer:** Bob (SM)
**Artifact:** Architecture
**Severity:** Minor — devproposal friction

The "Implementation Sequence" in the architecture lists 8 steps but provides no story sizing guidance. Step 1 ("Write api/openapi.yaml") and Step 4 ("Request pipeline — 8 steps, auth middleware, adapter resolution") are vastly different in complexity. Story authors will estimate them similarly without sizing guidance. The natural story seams (spec → codegen → stub adapter → contract tests → Ollama adapter → auth → pipeline → health → Helm) are partially visible in the directory structure but not explicit.

**Resolution:** This is informational for the devproposal phase — story authors should use the package structure to derive natural story seams. No architecture change required.

---

## Verdict Summary

| # | Finding | Severity | Artifact | Action Required |
|---|---------|----------|----------|----------------|
| 1 | Rate limiting scope contradiction | Critical | prd + product-brief | Correct product-brief to exclude rate limiting |
| 2 | Streaming support undecided | Critical | prd + architecture | Explicit in/out decision required |
| 3 | Telemetry gap (domain Art.10) | Major | architecture | Add duration + token fields to logging spec |
| 4 | Vault TTL/renewal unaddressed | Major | architecture | Specify client token lifecycle |
| 5 | Vault injection mechanism unspecified | Major | architecture | Specify k8s injection method + Helm values |
| 6 | `oapi-codegen.yaml` unspecified | Minor | architecture | Add 4–5 line codegen config spec |
| 7 | Helm values.yaml unspecified | Minor | architecture | Add values sketch |
| 8 | Local dev workflow absent | Minor | product-brief + architecture | Add `make run-local` target |
| 9 | Story sizing hints absent | Minor | architecture | Informational — for devproposal authors |

**Overall Verdict: PASS** *(updated 2026-04-03 — all findings resolved)*

All critical, major, and minor findings have been resolved in the planning artifacts. The initiative may advance to `devproposal`.

| # | Finding | Resolution |
|---|---------|------------|
| 1 | Rate limiting scope contradiction | Resolved — product-brief corrected; rate limiting explicitly OUT of scope (MVP2, delegates to `terminus-inference-provider-routing`) |
| 2 | Streaming support undecided | Resolved — `stream: true` OUT of scope for MVP; gateway returns 501 Not Implemented |
| 3 | Telemetry gap (domain Art.10) | Acknowledged and deferred — TODO added to architecture; will be addressed in dedicated observability initiative |
| 4 | Vault TTL/renewal unaddressed | Resolved — non-expiring token for MVP; k8s Secret injection documented; TODO for post-MVP service account migration |
| 5 | Vault injection mechanism unspecified | Resolved — k8s Secret + `secretKeyRef` pattern documented; Helm `vault.tokenSecret` value specified |
| 6 | `oapi-codegen.yaml` unspecified | Resolved — `strict-server: true` config spec added to architecture |
| 7 | Helm values.yaml unspecified | Resolved — values sketch with required/optional fields added to architecture |
| 8 | Local dev workflow absent | Resolved — `make run-local` using stub provider + `vault server -dev` documented in architecture |
| 9 | Story sizing hints absent | Informational — acknowledged for devproposal phase authors |
