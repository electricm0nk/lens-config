---
stepsCompleted: [step-01-init, step-02-discovery, step-02b-vision, step-02c-executive-summary, step-03-success, step-04-journeys, step-05-domain, step-06-innovation, step-07-project-type, step-08-scoping, step-09-functional, step-10-nonfunctional, step-11-polish]
inputDocuments:
  - docs/terminus/inference/gateway/product-brief.md
  - docs/terminus/platform/terminus-inference-service-plan.md
workflowType: prd
initiative: terminus-inference-gateway
classification:
  projectType: api_backend
  domain: AI infrastructure / internal platform
  complexity: high
  projectContext: greenfield service on brownfield platform
  userType: internal platform engineering consumers
  notes:
    - OpenAI compatibility is a distribution rationale — every LLM SDK already speaks it
    - Auth model: Vault token / k8s service account, not user identity — delegates to substrate
    - MVP scope: Ollama adapter (P1) working + adapter interface validated with stub P2
    - Success metric: zero contract renegotiation across full feature topology
  providerRoadmap:
    p1: Ollama (local, CPU, development, batch)
    p2: OpenAI API (cloud fallback, frontier model access)
    p3: Local GPU runtime (future, same adapter interface)
---

# Product Requirements Document - terminus-inference-gateway

**Author:** electricm0nk
**Date:** 2026-04-02

## Executive Summary

The `terminus-inference-gateway` establishes a stable, OpenAI-compatible API boundary in front of all inference runtimes on the terminus platform. It allows platform services to invoke AI capabilities through a single consistent endpoint without knowledge of which provider backend is active, enabling cost-driven local-first operation today and transparent migration to cloud or GPU runtimes as operational needs evolve.

**Target users:** Platform service authors and background job consumers within the terminus domain. Success is measured by zero contract renegotiation as the provider topology changes over time.

**Problem:** Without a stable gateway boundary, every terminus platform consumer couples directly to a specific inference backend. Provider changes, runtime additions, and fallback decisions propagate to all callers. The cost of this coupling grows with the number of consumers and the rate of infrastructure change.

**Motivation:** Local inference via Ollama has crossed the viability threshold for background and batch workloads. Running tasks locally eliminates token costs and rate limit exposure for unattended jobs that do not require frontier-model capability. The gateway is the abstraction layer that makes local-first the default without sacrificing the option to escalate to cloud when genuinely needed.

### What Makes This Special

The gateway's core value is **provider sovereignty without consumer lock-in**. OpenAI API compatibility is a distribution decision — every LLM client SDK in the ecosystem already speaks that dialect — not a vendor preference. A platform service targeting the gateway today can route through Ollama, OpenAI, or a future local GPU runtime by config change alone, with no code changes and no knowledge of what changed below.

The adapter interface is the primary deliverable of this feature. Ollama is the P1 reference implementation that validates the abstraction. The acceptance test for the feature is not "Ollama works" — it is "a second provider can be wired in with only config and a new adapter implementation."

Auth is delegated to the existing substrate: Vault-issued tokens or k8s service accounts. The gateway enforces authorization at the boundary without owning an identity store.

### Project Classification

| Dimension | Value |
|-----------|-------|
| **Type** | API backend — internal platform service |
| **Domain** | AI infrastructure |
| **Complexity** | High — abstraction discipline across provider churn |
| **Context** | Greenfield service on brownfield platform (k3s, Vault, terminus.platform) |
| **Provider roadmap** | Ollama (P1) → OpenAI API (P2) → Local GPU (P3) |

## Success Criteria

### User Success

**Primary — Unblocked operation:** A platform service author is never blocked from running work because of an external rate limit, quota exhaustion, or provider unavailability. Local-first as the default means the constraint is hardware, not a vendor's permission system.

**Secondary — Explicit failure modes:** Silent wrong responses are not tolerated. When the gateway cannot fulfil a request — provider unavailable, auth rejected, timeout exceeded, adapter error — it returns a structured, explicit error. Callers are never left guessing whether they got a degraded result or no result.

**Tertiary — Model flexibility by task class:** The gateway's adapter interface must carry sufficient context (route class, caller identity) to support task-based routing decisions in downstream features. Mundane or batch tasks route to cheaper, local, less-capable models. Advanced or latency-sensitive tasks are eligible for premium cloud models. The gateway does not own routing policy — that belongs to `terminus-inference-provider-routing` — but must be built to support it cleanly.

### Business / Operational Success

- Local inference handles the volume of routine platform workloads with zero cloud token spend for those tasks
- Cloud escalation is a deliberate, auditable policy decision — never an accidental default
- Provider changes do not require platform service authors to update their code

### Technical Success

- Contract test suite passes across all active provider adapters
- All error conditions produce structured responses — no silent failures, no swallowed exceptions
- A second provider adapter can be registered with config and a new adapter implementation only — no gateway handler changes
- Gateway delegates identity to Vault/k8s service account — no internal user store
- Health and readiness endpoints exist and are accurate

### Measurable Outcomes

| Outcome | Target |
|---------|--------|
| Provider swap requiring code change | 0 |
| Silent / ambiguous error responses | 0 |
| Cloud token spend for batch/mundane tasks | 0 (local-first default) |
| Contract test coverage for exposed routes | 100% of active routes |
| Second adapter stubbable without handler changes | Required before feature is done |

## User Journeys

### Journey 1 — Platform Service Author: Happy Path

**Alex** is building a new summarization step for the DailyBriefingWorkflow. The task is mundane — chunk a document, summarize each section, stitch the output. It runs nightly, unattended.

Alex configures the LLM client with the gateway URL and a Vault-issued service token. Model target: `ollama/mistral`. They send a `POST /v1/chat/completions` request. The response arrives in OpenAI chat completion format — the same schema Alex would get from any OpenAI-compatible SDK. The task works on the first try.

Alex never installed Ollama. Never managed a model pull. Never wrote a retry or fallback. The gateway resolved the provider, forwarded the request, normalized the response, and returned it. Alex moves on to the next story.

*Capabilities revealed: OpenAI-compatible endpoint, Ollama adapter, response normalization, Vault token auth.*

---

### Journey 2 — Platform Service Author: Provider Failure

The same summarization task runs at 2am. Ollama is unhealthy — a process crash earlier in the evening left it unresponsive.

Without the gateway, the task would hang on a socket timeout or receive a raw connection error with no actionable information. The batch orchestrator wouldn't know if it got a bad model response or a dead backend.

With the gateway: the health check on the Ollama adapter fails before the request is forwarded. The response is a structured `503`:

```json
{
  "error": {
    "type": "provider_unavailable",
    "provider": "ollama",
    "reason": "health_check_failed",
    "message": "Ollama adapter failed readiness check"
  }
}
```

The batch orchestrator receives a known error shape, logs it correctly, and the retry policy executes as designed. No silent bad output. No mystery. No tokens burned.

*Capabilities revealed: Provider health checking, structured error response contract, explicit error taxonomy.*

---

### Journey 3 — Operator: Adding a Second Provider

Todd wants to wire in the OpenAI API as P2 for advanced task routes. He writes an `OpenAIAdapter` class implementing the provider adapter interface. He adds it to the gateway config — provider name, base URL, auth env var reference. No gateway handler code changes.

He runs the contract test suite. It executes against both the Ollama adapter and the OpenAI adapter, validating request schema, response schema, error shape, and normalization guarantees. Both pass.

Existing callers targeting `ollama/mistral` are unaffected. A new caller can now target `openai/gpt-4o` through the same endpoint surface.

*Capabilities revealed: Provider adapter interface contract, config-driven provider registration, contract test suite covering multiple adapters.*

---

### Journey 4 — Operator: Auth Rejection at Boundary

A misconfigured platform service attempts to call the gateway. It was deployed without a Vault token injected into its environment — a setup error.

The gateway validates the Authorization header before resolving a provider or forwarding any request. No token present — the request is rejected immediately with a `401`:

```json
{
  "error": {
    "type": "unauthorized",
    "message": "Missing or invalid authorization token"
  }
}
```

No model call is made. No Ollama connection is opened. No tokens are burned. The operator sees the 401 in logs, diagnoses the missing Vault secret, and fixes the deployment config.

*Capabilities revealed: Auth enforcement at gateway boundary, pre-provider request validation, Vault/k8s service account identity delegation.*

---

### Journey Requirements Summary

| Journey | Capabilities Required |
|---------|----------------------|
| Happy path | OpenAI-compatible endpoint, Ollama adapter, response normalization, Vault auth |
| Provider failure | Provider health checking, structured error contract, error taxonomy |
| Second provider | Adapter interface, config-driven registration, multi-adapter contract tests |
| Auth rejection | Boundary auth enforcement, pre-forwarding validation, no identity store |

The following section defines the concrete API contract that fulfils these journeys.

## API Contract Reference

### Endpoint Specification

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/chat/completions` | POST | Primary inference endpoint, OpenAI-compatible request/response |
| `/healthz` | GET | Gateway liveness check |
| `/readyz` | GET | Provider adapter readiness check |

### Data Schemas

**Request:** OpenAI chat completion format (`model`, `messages`, `temperature`, `max_tokens`, etc.)

**Response:** OpenAI chat completion format — includes `finish_reason` passthrough (normalized: `stop`, `length`, `content_filter`) to allow callers to detect truncation without provider-specific handling

**Error envelope:**
```json
{
  "error": {
    "type": "<error_type>",
    "provider": "<provider_name>",
    "reason": "<machine_readable>",
    "message": "<human_readable>"
  }
}
```

### Error Code Taxonomy

| HTTP Status | `error.type` | Condition |
|-------------|-------------|-----------|
| 400 | `bad_request` | Malformed or missing required fields in request |
| 401 | `unauthorized` | Missing or invalid auth credential |
| 422 | `unsupported_model` | Requested model not available from any registered adapter |
| 503 | `provider_unavailable` | Provider health check failed before request forwarded |
| 504 | `provider_timeout` | Provider did not respond within gateway timeout threshold |

### Rate Limits

None at MVP. Single-tenant local deployment, Ollama-only. Rate limiting deferred to `terminus-inference-provider-routing`.

### Versioning

Endpoint surface locked to OpenAI API v1 schema. No gateway-internal versioning scheme for MVP — API contract stability is a non-negotiable constraint from the service plan. Version incompatibility is treated as a breaking change requiring a new feature initiative.

### API Documentation

Contract expressed as OpenAPI spec collocated with implementation. Spec is source-of-truth for contract tests across provider adapters.

## Project Scoping & Phased Development

### MVP Strategy & Philosophy

**MVP Approach:** Platform MVP — succeeds when a consuming service (e.g., DailyBriefingWorkflow) can call the gateway and get a normalized inference response without managing Ollama, provider auth, or error handling themselves. No volume targets, no external users. Done when the first dependent service ships without inference scaffolding.

**Resource Model:** Solo developer. The adapter interface design is the single highest-leverage upfront decision — getting it right before any provider is wired in prevents rework across every downstream adapter.

### MVP Feature Set (Phase 1 — this feature)

**Core User Journeys Supported:** Happy path, provider failure, auth rejection (Journeys 1, 2, 4)

**Must-Have Capabilities:**
- `POST /v1/chat/completions` — OpenAI-compatible request/response
- Ollama adapter (P1 reference implementation of adapter interface)
- Auth enforcement at gateway boundary (Vault token / k8s service account)
- Structured error contract for all failure modes (400, 401, 422, 503, 504)
- Health and readiness endpoints
- Adapter interface defined and documented — second provider is stubbable from day one
- OpenAPI spec collocated with implementation as source-of-truth for contract tests

**Out of Scope for this feature:**
- CPU runtime deployment and management (`terminus-inference-runtime-cpu` — parallel track)
- Route profiles and fallback policy (`terminus-inference-provider-routing` — next feature)
- Rate limiting
- Cloud provider adapter

### Post-MVP Phased Roadmap

**Phase 2 — `terminus-inference-provider-routing`:**
- Declarative route profiles by task class (interactive → cloud-first, batch → local-first)
- Cloud fallback policy with explicit escalation criteria
- OpenAI API adapter (P2)
- Per-caller rate guardrails

**Phase 3 — Expansion:**
- GPU runtime adapter (when hardware arrives — `terminus-inference-runtime-gpu`)
- Cost and latency telemetry per route class (`terminus-inference-telemetry-usage`)
- Model selection policy by task type and cost budget

### Risk Mitigation Strategy

**Technical — Adapter interface design:**
*Risk:* A poorly designed adapter interface requires reworking every downstream provider.
*Mitigation:* Interface must be validated against at least one stub P2 adapter before the feature is considered complete. The Ollama adapter alone is not sufficient validation.

**Technical — CPU batch throughput:**
*Risk:* Ollama on CPU may be too slow for the unattended batch use case, undermining the local-first premise.
*Mitigation:* Accepted at gateway layer by ensuring `finish_reason` and gateway timeout are propagated cleanly — callers can observe truncation and timeouts without provider-specific handling. Throughput validation is owned by `terminus-inference-runtime-cpu`, not this feature.

**Resource — Solo developer scope creep:**
*Risk:* Temptation to wire in routing policy or provider config in gateway handlers before `provider-routing` feature ships.
*Mitigation:* Hard boundary — no provider-specific routing policy in gateway handlers (non-negotiable constraint from service plan).

## Functional Requirements

### Request Handling

- **FR1:** A calling service can submit an OpenAI-compatible chat completions request and receive an OpenAI-compatible response
- **FR2:** The gateway normalizes provider response fields into the OpenAI chat completion schema before returning to the caller
- **FR3:** The gateway propagates `finish_reason` (normalized to `stop`, `length`, or `content_filter`) in every successful response
- **FR4:** Callers target a specific model using the standard OpenAI `model` field (e.g., `ollama/mistral`)

### Provider Abstraction

- **FR5:** The gateway routes inference requests to a registered provider adapter without exposing provider-specific details to callers
- **FR6:** An operator can register a new provider adapter by implementing the adapter interface and adding it to config — no handler code changes required
- **FR7:** The gateway validates provider adapter readiness before forwarding any request
- **FR8:** The adapter interface can be exercised with a stub implementation independent of a live provider

### Authentication & Authorization

- **FR9:** All requests must present a Vault token or k8s service account credential to be processed
- **FR10:** The gateway validates caller credentials before resolving a provider or forwarding any request
- **FR11:** Requests with missing or invalid credentials are rejected at the gateway boundary with no provider call made

### Error Contract

- **FR12:** All error responses use a structured envelope with `type`, `provider`, `reason`, and `message` fields
- **FR13:** Malformed or incomplete requests are rejected with a `400` before provider resolution
- **FR14:** Requests for models unavailable from any registered adapter return `422` with type `unsupported_model`
- **FR15:** Provider health check failures return `503` with type `provider_unavailable` identifying the failing provider
- **FR16:** Provider response timeouts return `504` with type `provider_timeout`
- **FR17:** Auth failures return `401` with type `unauthorized`

### Observability & Health

- **FR18:** The gateway exposes a liveness endpoint indicating process health
- **FR19:** The gateway exposes a readiness endpoint indicating provider adapter health
- **FR20:** The readiness endpoint reflects current health state of all registered adapters

### Contract & Interoperability

- **FR21:** The gateway API contract is expressed as an OpenAPI specification collocated with the implementation
- **FR22:** The OpenAPI spec is the source-of-truth for provider adapter contract tests
- **FR23:** Contract tests validate request schema, response schema, error shape, and normalization guarantees across all registered adapters

## Non-Functional Requirements

### Performance

- **NFR-P1:** Gateway overhead (excluding provider processing time) must not exceed 100ms per request under normal operating conditions
- **NFR-P2:** Gateway timeout threshold for provider responses is configurable; default value must be set conservatively for batch workloads (suggested: 5 minutes for unattended batch)
- **NFR-P3:** Health and readiness endpoint responses must complete within 1 second

### Security

- **NFR-S1:** All inter-service communication uses TLS — no plaintext credential or response data in transit
- **NFR-S2:** Auth credentials are never logged, included in error responses, or forwarded to provider adapters
- **NFR-S3:** Provider-internal error details (stack traces, internal hostnames, account info) are not exposed in gateway error responses — only normalized error envelopes are returned to callers
- **NFR-S4:** Vault token or k8s service account validation is performed on every request — no session caching or token reuse between requests

### Reliability

- **NFR-R1:** Provider failures are detected and returned as structured errors within the gateway timeout threshold — no silent hangs or raw socket errors exposed to callers
- **NFR-R2:** Gateway process must restart cleanly after a crash without requiring manual state cleanup
- **NFR-R3:** A provider adapter failure must not affect gateway availability for requests targeting other registered adapters

### Integration

- **NFR-I1:** The gateway depends on `terminus-infra-secrets` for Vault/k8s credential validation — this dependency must be declared and tested as a startup gate
- **NFR-I2:** The gateway depends on `terminus-infra-k3s` for deployment substrate — infrastructure readiness is a deployment prerequisite, not a runtime concern of the gateway
- **NFR-I3:** The Ollama adapter must be configurable via environment variable or config file — no hardcoded provider URLs or credentials in source
