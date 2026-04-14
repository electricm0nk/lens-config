---
stepsCompleted: [step-01-init, step-02-discovery, step-02b-vision, step-02c-executive-summary, step-03-success, step-04-journeys, step-05-domain, step-07-project-type, step-08-scoping, step-09-functional, step-10-nonfunctional, step-11-polish]
inputDocuments:
  - docs/terminus/inference/gateway/prd.md
  - docs/terminus/inference/gateway/architecture.md
  - docs/terminus/inference/gateway/tech-decisions.md
  - docs/terminus/inference/provider-routing/prd.md
workflowType: prd
initiative: terminus-inference-openaiadapter
classification:
  projectType: api_backend
  domain: AI infrastructure / internal platform
  complexity: medium
  projectContext: brownfield service on brownfield platform
  notes:
    - Adapter interface is pre-defined — complexity lives in secrets pipeline and deployment wiring
    - Definition of done: adapter returns non-error response via ESO-delivered Vault key in deployed gateway
    - Error propagation (429, 401) is an explicit requirement
    - Helm chart env var pattern must be confirmed before story scoping
---

# Product Requirements Document - terminus-inference-openaiadapter

**Author:** electricm0nk
**Date:** 2026-04-07

## Executive Summary

The `terminus-inference-openaiadapter` wires the OpenAI API into the terminus inference gateway as a second registered provider adapter, enabling cloud-based inference escalation when local Ollama capacity is unavailable. The immediate consumer is fourdogs central. The motivation is availability — with Ollama as the sole inference path, any local outage blocks all platform inference consumers with no fallback. This initiative closes that gap.

Platform consumers call the gateway's existing OpenAI-compatible endpoint. The gateway resolves which provider to call via the routing policy in `terminus-inference-provider-routing`. When the OpenAI adapter is selected, it forwards the request to the OpenAI API, normalises the response into the gateway's stable contract, and returns it — all invisible to the caller. No consumer code changes. No API key management by consumers. No awareness of which provider was used.

**Target users:** fourdogs central and any future terminus platform service that consumes the inference gateway.

**Definition of done:** The adapter returns a successful (non-error) response to a routed inference request in the deployed k3s gateway, using an API key delivered by ESO from Vault.

### What Makes This Special

The gateway adapter interface was the primary deliverable of the gateway initiative — designed from day one to accept a second provider with only config and a new implementation. This initiative is the proof that the abstraction holds under real conditions: a second provider, a real API, production secrets, live deployment. The failover becomes real. Callers never know it happened.

### Project Classification

| Dimension | Value |
|-----------|-------|
| **Type** | API backend — provider adapter extension with full deployment slice |
| **Domain** | AI infrastructure / internal platform |
| **Complexity** | Medium — adapter interface pre-defined; complexity in secrets pipeline and deployment wiring |
| **Context** | Brownfield — extending a live, deployed service |
| **Language** | Go |

## Success Criteria

### User Success

The OpenAI provider adapter is available as a registered gateway provider. Platform consumers (starting with fourdogs central) are not required to make any code changes to benefit. The capability exists and is consumable — specific fourdogs central consumer success will be defined in that initiative.

### Business / Operational Success

- Terminus platform inference has a working cloud fallback path for availability resilience
- Ollama outages no longer represent a total loss of inference capability for the platform
- OpenAI API key is managed through the existing Vault/ESO secrets infrastructure — no manual key distribution

### Technical Success

- `OpenAIAdapter` implements the provider adapter interface and passes contract tests
- Gateway responds to a request routed to the OpenAI adapter with a non-error response in the deployed k3s environment
- When Ollama is unavailable and OpenAI is configured as fallback in the routing config, the routing layer selects OpenAI and the caller receives a valid response with `degraded: true`
- OpenAI error codes (401, 429) propagate as structured gateway error envelopes
- API key is delivered to the gateway pod via External Secrets Operator from Vault — no hardcoded or manually injected credentials

### Measurable Outcomes

| Outcome | Target |
|---------|--------|
| Contract test suite passes with OpenAI adapter registered | 100% pass |
| Smoke test: non-error response from OpenAI adapter in deployed gateway | ✅ Pass |
| Resilience test: Ollama down → OpenAI selected → `degraded: true` response | ✅ Pass |
| API key source | Vault → ESO → pod env var only |

## Product Scope

### MVP

- `OpenAIAdapter` implementation against existing interface
- OpenAI entry in `routing-config.yaml` with priority and budget fields
- Vault secret for `GATEWAY_OPENAI_API_KEY`, ESO `ExternalSecret`, Helm chart wiring
- Contract test coverage for OpenAI adapter
- Smoke test and resilience test in deployed k3s gateway

### Post-MVP

- Per-model routing within the OpenAI adapter (e.g. gpt-4o vs o3 by workload class)
- Cost alerting when OpenAI daily budget approaches ceiling

### Vision

- GPU-local adapter (P3) using same interface, completing the three-provider topology

## User Journeys

### Journey 1: Platform Consumer (fourdogs central) — Happy Path / Invisible Failover

Alex is running a scheduled fourdogs central pipeline that calls the terminus inference gateway to classify product data. The pipeline has been running against Ollama without incident. One morning, Ollama is unavailable — memory pressure on the host, model not loaded. Without this initiative, Alex's pipeline fails with a connection error and the job stops.

With the OpenAI adapter registered and the routing config placing OpenAI as the fallback for the workload class, the routing layer detects Ollama is unavailable, selects OpenAI, and forwards the request. The gateway returns a normalized response with `degraded: true` and `fallback_provider: openai`. Alex's pipeline continues. The log shows the degradation signal. Alex sees it in the next morning's run report and notes the Ollama availability issue — but the job completed.

*Reveals requirements for:* provider health detection at routing time, `degraded: true` propagation through gateway response, routing config fallback chain with OpenAI entry.

### Journey 2: Platform Consumer — OpenAI Error Path

Alex's pipeline runs a request that hits the OpenAI adapter during a period when the API key has expired. The OpenAI API returns a 401. The adapter maps this to the gateway's structured error envelope: `type: unauthorized, provider: openai, reason: api_key_invalid`. Alex's pipeline receives a clean, structured error — not a raw HTTP response, not a panic, not a timeout. The error is loggable and actionable. Alex can see immediately what broke and why.

*Reveals requirements for:* 401/429 error mapping in `OpenAIAdapter`, structured error envelope propagation.

### Journey 3: Operator — Provisioning and Wiring

Todd adds the OpenAI API key to Vault under the gateway's secret path. He applies the ESO `ExternalSecret` manifest — it syncs the key into the gateway pod's environment as `GATEWAY_OPENAI_API_KEY`. He adds the OpenAI provider entry to `routing-config.yaml`: model target, budget ceiling, priority position. He redeploys. The gateway logs a startup line confirming the OpenAI adapter is registered. No other service touched.

*Reveals requirements for:* Vault secret path, ESO ExternalSecret manifest, `routing-config.yaml` OpenAI entry schema, Helm chart env var passthrough.

### Journey 4: Operator — Smoke Test

Todd runs the acceptance test: sends a request through the deployed gateway with the routing config set to select OpenAI. He receives a non-error response. He then stops Ollama and sends another request — the routing layer falls back to OpenAI, response returns with `degraded: true`. Both tests pass. The initiative is done.

*Reveals requirements for:* smoke test story, resilience test story, deployed-environment test tooling.

### Journey Requirements Summary

| Capability | Revealed By |
|------------|-------------|
| `OpenAIAdapter` implementation | Journeys 1, 2 |
| Routing config OpenAI entry (fallback chain) | Journey 1 |
| 401/429 error mapping | Journey 2 |
| Vault secret + ESO ExternalSecret | Journey 3 |
| Helm chart `extraEnv` / env var passthrough | Journey 3 |
| Gateway startup log for registered adapters | Journey 3 |
| Smoke test + resilience test | Journey 4 |

---

## Domain Requirements

### 5.1 Secrets & Key Hygiene (NFR-S2)

The OpenAI API key (`GATEWAY_OPENAI_API_KEY`) must never appear in logs, error responses, source code, or Helm values files. The delivery path is Vault → ESO ExternalSecret → pod environment variable. No adapter code should log or propagate the key value — error messages referencing auth failures may indicate key invalidity but must never echo the key itself.

### 5.2 AI Data Sovereignty (Terminus Art. 7)

Terminus infrastructure is local-first by constitutional mandate. Sending data to an external AI provider is a permitted exception under Art. 7, but it must be bounded. The routing policy enforces the boundary: only workloads with `workloadClass: interactive` are eligible for the OpenAI fallback chain. Batch workloads remain on Ollama exclusively. The PRD scope must document this boundary explicitly; the routing config entry must encode `allowedClasses: [interactive]`.

### 5.3 Telemetry (Terminus Art. 10)

Every request handled by the OpenAI adapter must emit a structured log entry containing:

| Field | Value |
|---|---|
| `route_id` | from routing context |
| `provider` | `openai` |
| `model` | target model name |
| `duration_ms` | wall-clock latency |
| `input_tokens` | from response metadata |
| `output_tokens` | from response metadata |
| `outcome` | `success` / `error` / `rate_limited` / `unauthorized` |
| `cost_tag` | optional — for budget tracking hooks |

This is a hard requirement per Art. 10 — non-negotiable, not an enhancement.

### 5.4 Art. 15 AI Safety Exception Artifact

Sending user data to an external AI provider (OpenAI) triggers the org-level Art. 15 AI safety exception gate. The 7 required artifacts are:

1. Decision statement (what data, to whom, why)
2. Threat analysis (data exposure, supplier lock-in, availability dependency)
3. Mitigations (data minimisation, fallback-first routing, key rotation procedure)
4. Expiry date (annual review required)
5. Named approver (must be human, on record)
6. Rollback plan (disable adapter, re-deploy with Ollama-only config)
7. Verification plan (how is the exception validated in practice)

**Scope decision:** The Art. 15 exception artifact is **in scope for this initiative**. It must be authored and committed before the initiative is promoted to `medium`. It will be tracked as a functional deliverable alongside the code artifacts.

### Domain Risk Register

| Risk | Severity | Mitigation |
|---|---|---|
| OpenAI API key leaked in logs/errors | High | Structured error mapping; key never logged |
| Sending batch data to external provider | High | Routing policy: `allowedClasses: [interactive]` enforced at config layer |
| No telemetry on external provider calls | Medium | Adapter emits structured log per Art. 10 |
| Art. 15 exception not filed before promotion | High | Gate check: exception artifact committed before medium promotion |

---

## API Backend Specific Requirements

### Adapter Interface Specification

The OpenAI adapter implements the provider adapter interface defined in the gateway codebase:

- **Interface:** `ProviderAdapter`
- **Method:** `Complete(ctx context.Context, req CompletionRequest) (CompletionResponse, error)`
- **Registration:** adapter registers by name `"openai"` at gateway startup
- **No HTTP surface exposed** — adapter is called in-process by the routing engine

### Authentication Model

- API key delivered via environment variable: `GATEWAY_OPENAI_API_KEY`
- Populated at runtime by ESO ExternalSecret (Vault → k8s Secret → pod env)
- Adapter reads `os.Getenv("GATEWAY_OPENAI_API_KEY")` — key never stored in struct fields post-request
- No inbound authentication required — all calls originate from the trusted routing engine

### Data Schemas

**Input (gateway-internal):**
```
CompletionRequest {
  RouteID       string
  WorkloadClass string   // must be "interactive" to reach this adapter
  Messages      []Message  // passed verbatim to OpenAI
  Model         string     // target model, e.g. "gpt-4o-mini"
}
```

**Output (gateway-internal):**
```
CompletionResponse {
  Content      string
  Provider     string    // "openai"
  Model        string
  InputTokens  int
  OutputTokens int
  Degraded     bool      // false for direct OpenAI calls
}
```

### Error Codes

| OpenAI HTTP Status | Mapped Error Type | Reason Field |
|---|---|---|
| 401 Unauthorized | `unauthorized` | `"invalid_api_key"` |
| 429 Too Many Requests | `rate_limited` | `"quota_exceeded"` |
| 5xx Server Error | `provider_error` | `"openai_unavailable"` |
| Timeout | `timeout` | `"deadline_exceeded"` |

All errors returned as structured envelope: `{type, provider, reason, message}`.

### Rate Limits

Rate limiting is not enforced within the adapter. Budget ceiling (token/cost cap) is managed by the routing layer's `BudgetTracker`. The adapter's responsibility is to map 429 responses to `rate_limited` — the routing engine will then mark this provider exhausted and skip it on subsequent requests.

### API Documentation

No public API docs required. Internal adapter contract is documented by:
- The `ProviderAdapter` interface definition in the gateway codebase
- This PRD (operational contract, error codes, auth model)
- Contract tests in the gateway test suite (inference Art. 2 gate)

---

## Project Scoping & Phased Development

### MVP Strategy & Philosophy

**MVP Approach:** Problem-solving MVP — minimum working implementation that resolves the availability gap (Ollama-only fallback) without touching consumers or downstream systems.
**Team:** 1 developer, 1 operator (secrets/k8s wiring), 1 compliance owner (Art. 15 artifact).

### MVP Feature Set (Phase 1 — this initiative)

**Core User Journeys Supported:** Journeys 1, 2, 3, 4 (all journeys in scope)

**Must-Have Capabilities:**
- `OpenAIAdapter` implementation against existing `ProviderAdapter` interface
- OpenAI entry in `routing-config.yaml` (priority, budget ceiling, `allowedClasses: [interactive]`)
- Vault secret + ESO `ExternalSecret` for `GATEWAY_OPENAI_API_KEY`
- Helm chart `extraEnv` passthrough confirmed and wired
- Contract test coverage for OpenAI adapter (inference Art. 2 gate)
- Smoke test: non-error response via ESO-delivered key in deployed gateway
- Resilience test: Ollama down → OpenAI fallback → `degraded: true` response
- Art. 15 AI safety exception artifact (7 items, committed before medium promotion)
- Structured telemetry log per request (Terminus Art. 10)

### Risk Mitigation Strategy

| Risk | Mitigation |
|---|---|
| Helm env var pattern differs from assumption | Confirm from existing gateway Helm chart before story scoping begins |
| Art. 15 artifact delays medium promotion | Author in parallel with code stories; gate enforced before promotion PR |
| OpenAI model target ambiguity | Default to `gpt-4o-mini`; configurable via `routing-config.yaml` |
| ESO sync delay blocks smoke test | Include ESO sync verification in operator smoke test checklist |

---

## Functional Requirements

### Provider Adapter

- **FR1:** The gateway can route inference requests to an OpenAI-backed provider via a registered `OpenAIAdapter`
- **FR2:** The `OpenAIAdapter` maps the gateway's internal `CompletionRequest` to an OpenAI `/v1/chat/completions` HTTP request
- **FR3:** The `OpenAIAdapter` maps the OpenAI response to the gateway's internal `CompletionResponse`, including `InputTokens` and `OutputTokens`
- **FR4:** The `OpenAIAdapter` maps a 401 response from OpenAI to a structured error envelope with `type: unauthorized`
- **FR5:** The `OpenAIAdapter` maps a 429 response from OpenAI to a structured error envelope with `type: rate_limited`
- **FR6:** The `OpenAIAdapter` maps 5xx responses to a structured error envelope with `type: provider_error`
- **FR7:** The `OpenAIAdapter` maps request timeouts to a structured error envelope with `type: timeout`

### Routing Configuration

- **FR8:** The operator can register the OpenAI provider in `routing-config.yaml` with a priority position, model target, and budget ceiling
- **FR9:** The routing engine selects the OpenAI provider as a fallback when higher-priority providers (Ollama) are unavailable or exhausted
- **FR10:** The routing engine restricts OpenAI provider selection to requests with `workloadClass: interactive` only
- **FR11:** The gateway response includes `degraded: true` and `fallback_provider: openai` when the OpenAI adapter services a request that was originally intended for Ollama

### Secrets & Key Delivery

- **FR12:** The operator can store the OpenAI API key in Vault under the gateway's secret path
- **FR13:** An ESO `ExternalSecret` syncs the Vault secret into the gateway pod's environment as `GATEWAY_OPENAI_API_KEY`
- **FR14:** The `OpenAIAdapter` reads the API key from the pod environment variable at runtime without it appearing in source code, config files, or logs

### Deployment & Wiring

- **FR15:** The gateway Helm chart passes `GATEWAY_OPENAI_API_KEY` into the pod environment via `extraEnv` configuration
- **FR16:** The gateway logs a startup confirmation that the OpenAI adapter is registered at boot time

### Observability & Telemetry

- **FR17:** The `OpenAIAdapter` emits a structured log entry per request containing: `route_id`, `provider`, `model`, `duration_ms`, `input_tokens`, `output_tokens`, `outcome`, and `cost_tag`
- **FR18:** The structured log entry `outcome` field reflects the result: `success`, `error`, `rate_limited`, or `unauthorized`

### Testing & Validation

- **FR19:** The contract test suite includes coverage for the `OpenAIAdapter` and passes as a gate (inference Art. 2)
- **FR20:** An operator can run a smoke test that produces a non-error response from the OpenAI adapter against the deployed gateway using the ESO-delivered key
- **FR21:** A resilience test validates that when Ollama is unavailable, the routing engine selects OpenAI and the gateway response contains `degraded: true`

### Compliance

- **FR22:** An Art. 15 AI safety exception artifact containing all 7 required items is committed to the repository before the initiative is promoted to `medium` audience
- **FR23:** Batch workloads are prohibited from routing to the OpenAI provider — the routing policy enforces `allowedClasses: [interactive]` at the config layer

---

## Non-Functional Requirements

### Performance

- **NFR-P1:** The adapter must not contribute more than 50ms of processing overhead beyond OpenAI API response time (network latency excluded)
- **NFR-P2:** Request context, including the API key reference, must not be retained in memory after the response is returned

### Security

- **NFR-S1:** `GATEWAY_OPENAI_API_KEY` must never appear in log output, error messages, stack traces, or HTTP responses at any log level
- **NFR-S2:** The API key must only be sourced from the pod environment variable — not from config files, Helm values files, or hardcoded in source
- **NFR-S3:** All communication with the OpenAI API must use TLS — no plaintext HTTP permitted
- **NFR-S4:** The Art. 15 exception artifact must be reviewed and renewed annually; the expiry date must be explicit in the artifact

### Reliability

- **NFR-R1:** On transient OpenAI errors (5xx, timeout), the adapter propagates the structured error to the routing engine — retry and fallback decisions belong to the routing layer, not the adapter
- **NFR-R2:** The adapter must not cache responses — every request is a fresh call

### Observability

- **NFR-O1:** Structured telemetry logs (FR17) must be emitted at INFO level, ensuring visibility in production log aggregation without requiring debug mode

### Data Governance

- **NFR-D1:** Only `workloadClass: interactive` requests may be forwarded to OpenAI — batch workload data must not leave local infrastructure (Terminus Art. 7)
- **NFR-D2:** User prompt content forwarded to OpenAI must not be logged by the adapter — telemetry captures metadata only (tokens, duration, outcome)
