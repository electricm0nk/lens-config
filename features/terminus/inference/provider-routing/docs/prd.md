---
stepsCompleted: [step-01-init, step-02-discovery, step-02b-vision, step-02c-executive-summary, step-03-success, step-04-journeys, step-05-domain, step-06-innovation, step-07-project-type, step-08-scoping, step-09-functional, step-10-nonfunctional, step-11-polish]
inputDocuments:
  - docs/terminus/inference/gateway/prd.md
  - docs/terminus/inference/gateway/product-brief.md
  - docs/terminus/inference/gateway/architecture.md
workflowType: prd
initiative: terminus-inference-provider-routing
classification:
  projectType: api_backend
  domain: AI infrastructure / internal platform
  complexity: high
  projectContext: brownfield
  deploymentPattern: integrated at gateway routing boundary
  callerIdentity: Vault token (established by gateway auth middleware)
  rateLimitingStorage: in-process sliding window (interface-backed; Redis addable at scale)
  notes:
    - Builds on live gateway with immutable Provider interface
    - Rate guardrails are a bundled deferral from gateway MVP, secondary concern
    - Fallback behavior: graceful degrade with degraded:true signal in normalized response
    - Go library pattern recommended; architecture doc owns the integration pattern
---

# Product Requirements Document — terminus-inference-provider-routing

**Author:** Todd
**Date:** 2026-04-06

## Executive Summary

`terminus-inference-provider-routing` is a policy layer that integrates with the terminus inference gateway at the routing boundary, resolving every inference request to the most cost-appropriate registered provider based on the caller's declared workload class (`interactive` vs `batch`) and the configured provider priority chain for that class.

**Without this layer:** Batch jobs and interactive tasks compete for the same inference quota with no policy separating them — paid provider credits exhaust at inopportune times, blocking interactive workloads that needed them. Adding a second provider requires every caller to explicitly choose between providers, coupling business logic to infrastructure.

When the preferred provider is unavailable or rate-limited, the layer degrades gracefully to the next eligible provider and surfaces a `degraded: true` signal in the normalized response — callers that care can observe and react; callers that don't can ignore. The gateway propagates these fields without modification.

Provider profiles are defined in YAML config, loaded at gateway startup. Per-caller rate guardrails (in-process sliding window, Vault token identity) are included here as a deferral from the gateway MVP — a secondary concern bundled with routing for delivery efficiency.

### What Makes This Special

The gateway solved provider-agnosticism — callers bind to one stable endpoint. Provider routing adds cost governance — the routing decision is encoded in policy, not caller code or gateway handlers. Workload class is a first-class concept: batch tasks that run unattended never consume cloud inference budget; interactive tasks are never silently downgraded without a degradation signal. These rules live here once, permanently, regardless of how many providers are added downstream.

### Project Classification

| Attribute | Value |
|-----------|-------|
| Project Type | `api_backend` — internal policy layer, integrated at gateway routing boundary |
| Domain | AI infrastructure / internal platform |
| Complexity | High — multi-provider fallback chains, declarative profile evaluation, rate guardrails |
| Project Context | Brownfield — builds on live gateway with immutable `Provider` interface |
| Caller Identity | Vault token identity established by gateway auth middleware |
| Rate Limiting Storage | In-process sliding window (interface-backed; Redis addable at scale) |

## Success Criteria

### User Success

- A platform service submits a request tagged `batch` and it routes to the local provider without any caller configuration — zero per-caller routing logic required
- An `interactive` request that falls back to a lower-capability provider is always accompanied by a `degraded: true` signal and `fallback_provider` field — no silent downgrades
- A new provider (e.g. OpenAI) can be registered by adding a YAML profile entry and restarting the gateway — zero changes to any downstream caller code

### Business Success

- Zero unplanned cloud credit consumption from batch workloads at MVP — enforced by dollar-budget guardrails, not by convention
- OpenAI adapter onboarded and routing correctly under declared budget without any platform service code changes
- At 3 months: no incident where a budget limit was exceeded because the routing layer failed to enforce it

### Technical Success

- Route resolution adds negligible latency overhead — target sub-millisecond (profile lookup is a map operation, not network call)
- Dollar budget per provider configured as a single value in YAML; routing layer converts to token-equivalent limit using declared `price_per_1k_tokens` at startup
- Budget exhaustion triggers graceful fallback to next eligible provider in chain — callers receive a valid response with `degraded: true`, never a 429 or unhandled error
- Rate guardrail state survives request bursts but resets on restart (in-process; acceptable for MVP single-replica deployment)

### Measurable Outcomes

| Outcome | Measure | Target |
|---------|---------|--------|
| Batch workload isolation | Batch requests reaching cloud provider | 0 when local available |
| Budget enforcement | Cloud spend above declared limit | 0 overage incidents |
| Caller decoupling | Caller code changes to add 2nd provider | 0 |
| Degradation visibility | Fallback responses without `degraded` signal | 0 |
| Routing overhead | p99 route resolution latency | < 1ms |

## Product Scope

### MVP — Minimum Viable Product

- Declarative route profiles by workload class (`interactive`, `batch`) — YAML config
- Two registered providers: Ollama (local) + OpenAI (cloud)
- Dollar-budget guardrail per provider — static `price_per_1k_tokens` + daily budget ceiling in config
- Budget exhaustion → graceful fallback with `degraded: true` in response
- `Router.Resolve(ctx, req) → ProviderName` interface consumed by gateway
- In-process sliding window rate state, Vault token caller identity
- Startup validation: malformed profile config causes gateway startup failure with descriptive error

### Growth Features (Post-MVP)

- Budget reset cadence configurable (daily / weekly / monthly)
- Per-caller budget overrides (some callers allocated more quota than others)
- Redis-backed rate state for multi-replica deployments
- Budget utilization observable via existing gateway telemetry
- Additional workload classes beyond `interactive` / `batch`

## User Journeys

### Journey 1 — Operator: Onboarding a Second Provider

Todd has been running inference through Ollama exclusively. The `DailyBriefingWorkflow` is live but he's hitting the limits of local CPU for anything that needs quality output fast. He signs up for an OpenAI API key and sets a daily cap — $5/day, enough to cover interactive workloads without risk of a surprise bill.

He opens `routing-config.yaml` and adds an OpenAI entry: declares `price_per_1k_tokens`, sets `daily_budget_usd: 5`, and positions it as the preferred provider for `interactive` traffic with Ollama as fallback. Ollama remains the sole provider for `batch`. He restarts the gateway. The routing layer loads at startup, validates the config, and logs a clean startup line. No downstream caller touched.

The next morning, `DailyBriefingWorkflow` runs its batch jobs overnight — every one goes to Ollama. An interactive query during the day goes to OpenAI, comes back fast with quality output. Todd checks logs — routing decisions are visible, no surprises. He's onboarded a paid provider without writing a line of application code.

**Reveals requirements for:** profile YAML schema, startup validation, routing decision logging

### Journey 2 — Operator: Budget Exhaustion Handled Gracefully

Mid-afternoon, an interactive workload has been running heavier than usual. The routing layer's in-process counter hits the daily $5 OpenAI budget ceiling. The next interactive request arrives — the router checks OpenAI, sees budget exhausted, selects Ollama from the fallback chain. The gateway returns a normalized response with `degraded: true` and `fallback_provider: ollama`. The caller continues without error.

Todd sees the degradation in logs, notes the budget was hit earlier than expected. He can either raise the cap or let it ride — no incident, no failed requests, no manual intervention required.

**Reveals requirements for:** token budget accounting, budget-exhausted fallback, `degraded` response field, logging of budget state

### Journey 3 — Operator: Misconfiguration Caught at Startup

Todd is adding a third provider profile and makes a typo — `price_per_1k_tokens` is missing from the entry. He restarts the gateway. The routing layer startup validation catches the malformed profile and exits with a descriptive error naming the exact field and profile. The gateway never enters a running state with incomplete config. Todd fixes the YAML, restarts cleanly.

**Reveals requirements for:** startup config validation with field-level error messages

### Journey Requirements Summary

| Capability | Revealed By |
|-----------|-------------|
| YAML route profile schema (workload class → provider priority chain) | Journey 1 |
| Per-provider `price_per_1k_tokens` + `daily_budget_usd` config | Journey 1, 2 |
| Startup config validation with descriptive errors | Journey 3 |
| `Router.Resolve` selecting provider based on workload class | Journey 1, 2 |
| In-process token budget counter per provider | Journey 2 |
| Budget-exhausted graceful fallback with `degraded: true` | Journey 2 |
| Routing decision + budget state logging | Journey 1, 2 |

## Domain Requirements

### Technical Constraints

- **Cost accounting:** Budget tracking uses *actual* token counts from provider response (`usage.prompt_tokens` + `usage.completion_tokens`), not pre-request estimates. Conversion formula: `(prompt_tokens + completion_tokens) / 1000 * price_per_1k_tokens`. Tracking is cumulative per provider per reset period.
- **API key security:** Provider API keys must not appear in routing profile YAML. Keys are referenced by environment variable name in config (e.g. `api_key_env: OPENAI_API_KEY`) and resolved at startup. No credential values logged.
- **Data residency:** Request content (prompt, response) never passes through the routing layer — only metadata (workload class, token counts, caller identity) is observed. Routing is a pre-dispatch decision; the gateway forwards the request payload directly to the selected provider.
- **No prompt/response logging:** Routing logs record provider selected, workload class, token count, and degradation state only. No prompt or response content in routing logs.

### Risk Mitigations

| Risk | Mitigation |
|------|-----------|
| Token count unavailable in response | Fall back to estimated count from request body character length; log warning |
| Provider pricing change | Update `price_per_1k_tokens` in config and restart; no code change required |
| Budget counter drift on restart | Acceptable at MVP — counter resets on restart; noted in runbook |

## API Surface

`terminus-inference-provider-routing` exposes a Go interface consumed exclusively by the gateway. No HTTP endpoints. No SDK. The types and signatures below are informative — the architecture document owns the final integration pattern and package structure.

### Router Interface

```go
type Router interface {
    // Resolve selects the best available provider for the request.
    // Returns the provider name, or error if no eligible provider is available.
    Resolve(ctx context.Context, req *RoutingRequest) (string, error)

    // RecordUsage records actual token spend after a provider response is received.
    // Called by the gateway after every successful provider call.
    RecordUsage(ctx context.Context, provider string, promptTokens, completionTokens int)

    // HealthCheck returns per-provider status for inclusion in /readyz.
    // Never returns an error — unavailable providers are reported in the status map.
    HealthCheck(ctx context.Context) HealthStatus
}

// HealthStatus reports the routing layer's operational state.
type HealthStatus struct {
    Healthy   bool                      // false if no providers are eligible for any class
    Providers map[string]ProviderStatus // keyed by provider name
}

type ProviderStatus struct {
    BudgetUsedUSD   float64 // cumulative spend since last reset
    BudgetLimitUSD  float64 // 0 = unlimited
    BudgetExhausted bool
}

// RoutingRequest carries the workload context needed to make a routing decision.
type RoutingRequest struct {
    WorkloadClass string // "interactive" or "batch"
    CallerIdentity string // Vault token identity, established by gateway auth middleware
}
```

### Startup

```go
// LoadProfiles loads and validates routing profiles from the given YAML path.
// Returns a configured Router on success, fatal error with field-level details on failure.
func LoadProfiles(path string) (Router, error)
```

### Config Schema

```yaml
schema_version: 1

providers:
  - name: ollama
    api_key_env: ""            # empty = no auth required
    price_per_1k_tokens: 0.0   # local; no cost
    daily_budget_usd: 0        # 0 = unlimited

  - name: openai
    api_key_env: OPENAI_API_KEY
    price_per_1k_tokens: 0.002
    daily_budget_usd: 5.00

profiles:
  interactive:
    priority: [openai, ollama]  # first available within budget
  batch:
    priority: [ollama, openai]  # local-first; cloud only if local unavailable
```

### Error Codes

| Code | Condition |
|------|-----------|
| `no_eligible_provider` | All providers in the priority chain are either over budget or unavailable |
| `invalid_workload_class` | Request carries an unrecognized workload class not defined in any profile |
| `config_load_error` | Profile YAML is malformed or missing required fields (startup only) |

### Degradation Response Fields

When fallback occurs, the gateway propagates these fields in the normalized response (routing layer sets them in context):

| Field | Type | Description |
|-------|------|-------------|
| `degraded` | bool | `true` when a fallback provider was used |
| `fallback_provider` | string | Name of the provider that actually handled the request |
| `degradation_reason` | string | `budget_exhausted` \| `provider_unavailable` |

## Functional Requirements

### Routing Policy — Core

**FR1:** The gateway can resolve an inference request to a provider name by passing workload class and caller identity to the router — without any routing logic in gateway handlers.

**FR2:** The router selects a provider by evaluating the provider priority chain configured for the request's workload class.

**FR3:** The router supports at minimum two workload classes: `interactive` (cloud-first) and `batch` (local-first).

**FR4:** Each workload class has an independently configurable provider priority chain.

**FR5:** When the first-priority provider is budget-exhausted or unavailable, the router selects the next eligible provider in the chain.

**FR6:** When fallback occurs, the router surfaces `degraded: true`, `fallback_provider`, and `degradation_reason` in the response context for the gateway to propagate.

**FR7:** When no eligible provider exists in the chain, the router returns a `no_eligible_provider` error — no silent routing to an unexpected provider.

### Budget Guardrails

**FR8:** The operator can declare a daily dollar budget per provider in the routing config.

**FR9:** The router converts the dollar budget to a token limit at startup using the declared `price_per_1k_tokens` for that provider.

**FR10:** The gateway can record actual token spend after each provider response by calling `RecordUsage` with prompt and completion token counts.

**FR11:** The router tracks cumulative token spend per provider in-process and marks a provider as budget-exhausted when the token limit is reached.

**FR12:** A provider with `daily_budget_usd: 0` is treated as unlimited — no budget tracking performed.

**FR13:** Budget state resets on process restart. Persistence across restarts is not required at MVP.

### Config & Startup

**FR14:** The operator can define all routing policy (providers, profiles, budgets) in a single YAML file with a declared `schema_version`.

**FR15:** Provider API keys are referenced by environment variable name in config — no plaintext credentials in the YAML file.

**FR16:** The routing layer validates the full config at startup and exits with a field-level error message if any required field is missing or invalid.

**FR17:** A successful config load is confirmed by a structured startup log line.

### Observability

**FR18:** The router logs the selected provider, workload class, and whether degradation occurred for every resolved request.

**FR19:** Routing logs never include prompt content, response content, or credential values.

**FR20:** The router exposes a `HealthCheck` method returning per-provider budget state (`budget_used_usd`, `budget_limit_usd`, `budget_exhausted`) for inclusion in the gateway's `/readyz` response.

**FR21:** The gateway's `/readyz` reports the routing layer as unhealthy if no providers are eligible for any configured workload class.

### Extensibility

**FR22:** A new provider can be added to the routing layer by adding a provider entry and updating profile priority chains in config — no routing layer code changes required.

**FR23:** A new workload class can be added by adding a new profile entry in config — no routing layer code changes required.

## Non-Functional Requirements

### Performance

**NFR-P1:** Route resolution (`Resolve`) completes in < 1ms p99 under normal operating conditions — it is a map lookup plus a counter check, not a network call.

**NFR-P2:** `RecordUsage` is non-blocking — it must not add measurable latency to the response path. Implemented as an in-process counter update; no I/O permitted.

**NFR-P3:** `HealthCheck` completes in < 5ms — reads in-memory state only, no provider network calls.

### Security

**NFR-S1:** No API key or credential values appear in any log output, error message, or `HealthStatus` response.

**NFR-S2:** No prompt content, response content, or caller identity tokens appear in routing logs — only opaque identifiers and metadata.

**NFR-S3:** API keys are resolved from environment variables at startup only — never read from config values at runtime.

**NFR-S4:** The routing layer does not perform its own authentication — caller identity is consumed from the request context established by the gateway auth middleware.

### Integration

**NFR-I1:** The `Router` interface is the sole integration boundary between the gateway and the routing layer — no direct package dependencies beyond the interface and its types.

**NFR-I2:** The routing layer must compile cleanly against the Go version used by the gateway — no separate Go version requirements.

**NFR-I3:** The routing layer introduces no transitive dependencies that conflict with existing gateway dependencies.

### Reliability

**NFR-R1:** A panic in the routing layer must not crash the gateway process — the router must recover panics internally and return a `no_eligible_provider` error.

**NFR-R2:** A missing or unrecognized workload class in an incoming request never causes a panic — it returns `invalid_workload_class` error cleanly.

**NFR-R3:** If all providers become budget-exhausted, the router returns `no_eligible_provider` — it never silently routes to an unintended provider or blocks indefinitely.
