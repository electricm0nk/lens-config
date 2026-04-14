---
stepsCompleted: [step-01-init, step-02-context, step-03-starter, step-04-decisions, step-05-patterns, step-06-structure, step-07-validation, step-08-complete]
status: complete
completedAt: '2026-04-06'
inputDocuments:
  - docs/terminus/inference/provider-routing/prd.md
  - docs/terminus/inference/gateway/architecture.md
workflowType: architecture
initiative: terminus-inference-provider-routing
project_name: terminus-inference-provider-routing
date: '2026-04-06'
---

# Architecture: Terminus Inference Provider Routing

_This document builds collaboratively through step-by-step discovery. Sections are appended as we work through each architectural decision together._

---

## Project Context

**Initiative:** terminus-inference-provider-routing
**Type:** Go library package — `internal/routing/` inside the `terminus-inference-gateway` module
**Brownfield context:** Extends the existing gateway; must comply with its `internal/` conventions, `Provider` interface contract, `slog` logging, and `context.Context` discipline.

**Input documents:**
- `docs/terminus/inference/provider-routing/prd.md` — 23 FRs, 10 NFRs, Router interface contract, config schema, error codes
- `docs/terminus/inference/gateway/architecture.md` — immutable `Provider` interface, package conventions, error wrapping patterns

**Key constraints from gateway:**
- `Provider` interface is immutable: `Chat(ctx, *ChatRequest) (*ChatResponse, error)`, `Health(ctx) error`, `Name() string`
- All packages live under `internal/` — no exported packages except via router entry point
- Errors wrapped as `fmt.Errorf("component: action: %w", err)`
- Structured logging via `slog` — no global logger, logger injected
- `context.Context` always propagated from inbound request, never `context.Background()` in hot path

---

## Starter Evaluation

**Decision:** Go library package inside the existing gateway module — no new repo, no new Go module.

**Rationale:**
- Single-replica deployment; startup-loaded config; solo developer — separate repo adds friction with no benefit
- `internal/routing/` enforces encapsulation: only the gateway's HTTP handlers can import it
- Keeps dependency graph simple; provider-routing changes and gateway changes ship in the same commit/PR when needed

**No framework selected** — stdlib only (`sync`, `encoding/yaml`, `os`, `errors`, `log/slog`). YAML unmarshalling via `gopkg.in/yaml.v3` (already a gateway dependency or straightforward to add).

---

## Core Architectural Decisions

### Decision 1: Budget Tracker Concurrency — `sync.RWMutex`

**Choice:** `sync.RWMutex` per provider, held by `BudgetTracker`.

**Rationale:** `Resolve` is read-heavy (many concurrent inbound requests check budget status); `RecordUsage` is write-infrequent (one write per completed response). RWMutex correctly models this access pattern. Simple, correct, no external dependencies, easy to test with `-race`.

**Rejected alternatives:**
- `sync/atomic` — insufficient for struct-level budget state (multiple fields)
- Separate goroutine + channel — unnecessary complexity for this access pattern

### Decision 2: Config Path Injection — `ROUTING_CONFIG_PATH` env var

**Choice:** `os.Getenv("ROUTING_CONFIG_PATH")` in `LoadProfiles`; path can also be passed explicitly.

**Rationale:** Consistent with gateway's env-var config pattern. No flags package dependency. Testable by setting env in test or by calling `LoadProfiles(path)` directly with an explicit path.

**Config format:** YAML with top-level `schema_version` field. Unsupported version = hard error at load time.

### Decision 3: Error Sentinel — `ErrNoEligibleProvider`

**Choice:** `("", ErrNoEligibleProvider)` returned from `Resolve` when no provider is eligible (all exhausted, all unhealthy, or no profile match).

**Rationale:** Sentinel errors compose correctly with `errors.Is` unwrapping even when wrapped with context. Gateway's `internal/errors` maps this to HTTP 503. Callers never inspect error strings.

**Additional sentinel:** `ErrBudgetExhausted` — returned (wrapped) when a specific provider is selected but over budget; `Resolve` falls through to next eligible provider before returning `ErrNoEligibleProvider` if all are exhausted.

---

## Implementation Patterns & Consistency Rules

### File Layout

One file per major responsibility:
- `router.go` — `Router` interface + `router` struct + `Resolve` / `HealthCheck`
- `budget.go` — `BudgetTracker` struct + `RecordUsage` + budget exhaustion check
- `config.go` — `LoadProfiles`, config structs, YAML unmarshalling
- `errors.go` — exported sentinel errors (`ErrNoEligibleProvider`, `ErrBudgetExhausted`)
- `types.go` — `RoutingRequest`, `RoutingResponse`, `HealthStatus`, other shared types
- `*_test.go` — co-located with each source file, same package (`package routing`)

### Error Handling

- `Resolve` wraps sentinel with profile context:
  `fmt.Errorf("routing: no eligible provider for profile %q: %w", profile, ErrNoEligibleProvider)`
- Callers use `errors.Is(err, ErrNoEligibleProvider)` — never string-match
- `LoadProfiles` wraps YAML/IO errors: `fmt.Errorf("routing: load config %q: %w", path, err)`
- Internal errors escalate upward; no silent swallowing

### Logging

All logging via `slog` structured logger. Logger injected at construction (not global).

**Allow-list (log these):**
- Provider name selected by `Resolve`
- Profile name from request
- Token counts (prompt, completion) in `RecordUsage`
- Budget state transitions (OK → exhausted, exhausted → reset)
- Config load success/failure (path only, not content)

**Never log:**
- Prompt text, message content, response text
- API keys, auth tokens, request headers
- User identifiers or session data

Log levels: `Debug` for `Resolve` selection, `Info` for budget transitions, `Error` for config failures.

### Concurrency

- `BudgetTracker` holds one `sync.RWMutex` per provider
- Reads (`Resolve` budget check): `mu.RLock(); defer mu.RUnlock()`
- Writes (`RecordUsage`): `mu.Lock(); defer mu.Unlock()`
- `defer` unlock immediately after lock acquisition — no manual paired calls
- `Resolve` never acquires a write lock

### Config Zero-Value Semantics

- `budget_limit: 0` → unlimited (feature disabled for that provider)
- `budget_limit: -1` (or negative) → config validation error at `LoadProfiles` time
- Missing optional fields → zero value = feature disabled, not an error
- `schema_version` field required; unsupported version → hard error at load time

### Testing

- Table-driven tests for all multi-case logic; single test function per behavior area
- Co-located `_test.go` files, `package routing` (white-box)
- No `tests/` or `testdata/` subdirectory unless fixtures require files
- Mock providers implement the gateway `Provider` interface — no test-only interfaces
- Budget tests use deterministic token counts; no real HTTP calls

---

## Project Structure & Boundaries

### Complete Directory Structure

```
terminus-inference-gateway/           # existing gateway module root
│
├── internal/
│   ├── routing/                      # NEW package (this feature)
│   │   ├── router.go                 # Router interface; router struct; Resolve, HealthCheck
│   │   ├── budget.go                 # BudgetTracker; RecordUsage; budget exhaustion check
│   │   ├── config.go                 # LoadProfiles; RoutingConfig struct; YAML unmarshal; schema validation
│   │   ├── errors.go                 # ErrNoEligibleProvider; ErrBudgetExhausted
│   │   ├── types.go                  # RoutingRequest; RoutingResponse; HealthStatus; ProviderStatus
│   │   ├── router_test.go            # Resolve table-driven tests; HealthCheck tests
│   │   ├── budget_test.go            # RecordUsage concurrency tests; exhaustion threshold tests
│   │   └── config_test.go            # LoadProfiles happy path; schema version rejection; zero-value semantics
│   │
│   ├── errors/                       # EXISTING — maps routing.ErrNoEligibleProvider → HTTP 503
│   ├── providers/                    # EXISTING — Provider interface + OpenAI/Ollama implementations
│   └── server/                       # EXISTING — HTTP handler; imports internal/routing
│
├── cmd/
│   └── gateway/
│       └── main.go                   # EXISTING — calls routing.LoadProfiles; injects Router into server
│
└── docs/
    └── terminus/inference/
        ├── gateway/                  # EXISTING — gateway prd/architecture
        └── provider-routing/         # NEW
            ├── prd.md                # complete
            └── architecture.md       # this document
```

### FR → File Mapping

| FR group | FR IDs | File |
|---|---|---|
| Profile resolution | FR1–FR5 | `router.go`, `types.go` |
| Budget tracking | FR6–FR10 | `budget.go` |
| Config / health | FR11–FR15 | `config.go`, `router.go` |
| Observability / security | FR16–FR19 | cross-cutting (slog allow-list in all files) |
| Error handling / degradation | FR20–FR23 | `errors.go`, `router.go` |

### Architectural Boundaries

**Package boundary**
`internal/routing/` is importable only by other packages within `terminus-inference-gateway`. External modules cannot import it — enforced by Go's `internal/` directory rule.

**Interface boundary**
All gateway code interacts with `Router` (interface), never the unexported `router` struct. `LoadProfiles` returns `(Router, error)`. Tests use the same interface — no test-only interfaces.

**Config boundary**
`LoadProfiles` is the single entry point for all configuration. Callers never construct `RoutingConfig` structs directly. Schema validation and zero-value normalization happen inside `LoadProfiles` before a `Router` is returned.

**Budget boundary**
`BudgetTracker` is unexported. Only `router.RecordUsage` writes to it; only `router.Resolve` reads from it for eligibility checks. No caller touches budget state directly.

### Integration Points

**Startup flow:**
```
main.go
  → routing.LoadProfiles(os.Getenv("ROUTING_CONFIG_PATH"))
  → (Router, error)
  → injected into server/handler
```

**Per-request flow:**
```
HTTP handler
  → router.Resolve(ctx, &RoutingRequest{Profile: "..."}) → (providerName, error)
  → errors.Is(err, routing.ErrNoEligibleProvider) → 503 via internal/errors
  → providers[providerName].Chat(ctx, req) → response
  → router.RecordUsage(ctx, providerName, usage.PromptTokens, usage.CompletionTokens)
```

**Health check flow:**
```
HTTP GET /health
  → router.HealthCheck(ctx) → HealthStatus
  → per-provider status included in health response
```

**Error mapping (additions to existing `internal/errors`):**
```go
case errors.Is(err, routing.ErrNoEligibleProvider):
    return http.StatusServiceUnavailable, "no eligible provider"
case errors.Is(err, routing.ErrInvalidWorkloadClass):
    return http.StatusBadRequest, "invalid workload class"
```

---

## Architecture Validation

### FR Coverage

All 23 FRs are architecturally supported. Four gaps identified during validation and resolved below.

### NFR Coverage

All 10 NFRs are architecturally supported. Two gaps identified during validation and resolved below.

### Gap Resolutions

**Gap 1 — `ErrInvalidWorkloadClass` sentinel (NFR-R2)**

`errors.go` exports three sentinels:
```go
var (
    ErrNoEligibleProvider  = errors.New("routing: no eligible provider")
    ErrBudgetExhausted     = errors.New("routing: budget exhausted")
    ErrInvalidWorkloadClass = errors.New("routing: invalid workload class")
)
```
`Resolve` returns `("", ErrInvalidWorkloadClass)` when the profile name in `RoutingRequest` is not present in the loaded config. Gateway maps this to HTTP 400 (bad request), distinct from `ErrNoEligibleProvider` → 503.

**Gap 2 — Panic recovery in `Resolve` (NFR-R1)**

`router.go` `Resolve` wraps its body in a named-return recover:
```go
func (r *router) Resolve(ctx context.Context, req *RoutingRequest) (provider string, err error) {
    defer func() {
        if rec := recover(); rec != nil {
            slog.ErrorContext(ctx, "routing: panic in Resolve", "recover", rec)
            provider, err = "", ErrNoEligibleProvider
        }
    }()
    // ... resolution logic
}
```
No other method requires panic recovery — `RecordUsage` operates on in-memory counters only, `HealthCheck` reads state only.

**Gap 3 — `RoutingResponse` type specification (FR6)**

`types.go` defines:
```go
type RoutingResponse struct {
    Provider         string // selected provider name
    Degraded         bool   // true if a fallback provider was selected
    FallbackProvider string // non-empty when Degraded == true
    DegradationReason string // "budget_exhausted" | "provider_unhealthy"
}
```
`Resolve` returns `(string, error)` per the PRD interface — the `RoutingResponse` is populated by a separate value in an extended signature or returned as a second return value. **Resolution:** `Resolve` returns `(*RoutingResponse, error)` where `RoutingResponse.Provider` carries the selected name and degradation fields are populated inline. The PRD interface signature (`(string, error)`) is informative; the architecture owns the final signature.

Updated `Router` interface:
```go
type Router interface {
    Resolve(ctx context.Context, req *RoutingRequest) (*RoutingResponse, error)
    RecordUsage(ctx context.Context, provider string, promptTokens, completionTokens int)
    HealthCheck(ctx context.Context) HealthStatus
}
```

**Gap 4 — API key resolution mechanism (FR15)**

Config stores the *name* of the environment variable, not the key value. `LoadProfiles` resolves at startup:
```go
// in config.go, during LoadProfiles:
apiKey := os.Getenv(providerCfg.APIKeyEnv)
if apiKey == "" {
    return nil, fmt.Errorf("routing: load config %q: provider %q: env var %q is not set", path, providerCfg.Name, providerCfg.APIKeyEnv)
}
```
Resolved keys are stored in an unexported field of the provider's runtime entry — never in a type that appears in logs or `HealthStatus`.

**Gap 5 (minor) — FR13 explicitness**

Budget state is in-process only. It is not persisted to disk, a database, or any external store. Restarting the gateway process resets all budget counters to zero. Persistence across restarts is explicitly out of scope for MVP.

**Gap 6 — ROUTING_CONFIG_PATH not-set behavior (adversarial review F5)**

If `ROUTING_CONFIG_PATH` is empty, `LoadProfiles` returns an error immediately:
```go
if path == "" {
    return nil, errors.New("routing: ROUTING_CONFIG_PATH environment variable is not set")
}
```
No default path is attempted.

**Gap 7 — Schema version (adversarial review F6)**

`schema_version: 1` is the only supported value at MVP.
```go
const SupportedSchemaVersion = 1
```
If the YAML file declares any other value, `LoadProfiles` returns a hard error naming the field and the supported value.

**Gap 8 — Canonical YAML config schema (adversarial review F7)**

```yaml
schema_version: 1

providers:
  - name: ollama
    base_url: http://localhost:11434
    api_key_env: OLLAMA_API_KEY        # env var name; "" is valid for unauthenticated
    price_per_1k_tokens: 0.0
    daily_budget_usd: 0                # 0 = unlimited

  - name: openai
    base_url: https://api.openai.com/v1
    api_key_env: OPENAI_API_KEY
    price_per_1k_tokens: 0.002
    daily_budget_usd: 5.00

profiles:
  - name: interactive
    provider_priority: [openai, ollama]  # first eligible wins

  - name: batch
    provider_priority: [ollama, openai]
```

**Gap 9 — HealthStatus and ProviderStatus shapes (adversarial review F9)**

```go
// in types.go

type ProviderStatus struct {
    Name            string
    BudgetLimitUSD  float64
    BudgetUsedUSD   float64
    BudgetExhausted bool
}

type HealthStatus struct {
    Providers  []ProviderStatus
    Degraded   bool   // true if any configured provider is budget-exhausted
    AllExhausted bool  // true if ALL providers are budget-exhausted (maps to /readyz unhealthy)
}
```

**Gap 10 — Degradation reason constants (adversarial review F10)**

```go
// in types.go or errors.go
const (
    DegradationReasonBudgetExhausted = "budget_exhausted"
    DegradationReasonUnhealthy       = "provider_unhealthy"
)
```

**Gap 11 — Caller identity in RoutingRequest (adversarial review F2)**

Caller identity is available in `context.Context` (established by gateway auth middleware, per NFR-S4) but is NOT included in `RoutingRequest` and is NOT used for provider selection in MVP. Rate-guardrail features that would use caller identity are explicitly deferred.

**Gap 12 — Budget reset anchor (adversarial review F3)**

For MVP, "daily" budget is process-start-relative. There is no clock-based reset — the budget counter resets only when the gateway process restarts. This is consistent with FR13 (no persistence). A production deployment should be aware that a long-running process accumulates budget until restarted.

**Gap 13 — Token fallback formula (adversarial review F4)**

When `usage` is nil in the provider response, `RecordUsage` uses the estimate:
```go
estimatedTokens := len(requestBodyJSON) / 4  // standard 4-chars-per-token heuristic
```
And logs:
```go
slog.WarnContext(ctx, "routing: usage missing from response, using estimate",
    "provider", provider, "estimated_tokens", estimatedTokens)
```

**Gap 14 — testdata directory (adversarial review F11)**

`config_test.go` uses YAML fixture files located at `internal/routing/testdata/*.yaml`. These files are the canonical source for config schema examples in tests.

**Gap 15 — Race flag requirement (adversarial review F12)**

All tests in `internal/routing/` MUST be run with `-race`. CI must include: `go test -race ./internal/routing/...`

### Architecture Completeness Checklist

**Requirements Analysis**
- [x] Brownfield gateway constraints identified and respected
- [x] 23 FRs mapped to specific files
- [x] 10 NFRs each addressed by concrete architectural choice
- [x] Cross-cutting concerns (logging, concurrency, error handling) mapped

**Architectural Decisions**
- [x] Concurrency model: `sync.RWMutex` per provider
- [x] Config injection: `ROUTING_CONFIG_PATH` env var
- [x] Error sentinels: `ErrNoEligibleProvider`, `ErrBudgetExhausted`, `ErrInvalidWorkloadClass`
- [x] Module layout: `internal/routing/` in gateway repo
- [x] No new external dependencies beyond `gopkg.in/yaml.v3`

**Implementation Patterns**
- [x] File-per-responsibility layout defined
- [x] Error wrapping convention established
- [x] Logging allow/never-log lists defined
- [x] Mutex discipline rules stated
- [x] Config zero-value semantics stated
- [x] Test organization conventions defined

**Project Structure**
- [x] Complete directory tree with per-file responsibility
- [x] FR → file mapping table
- [x] Package, interface, config, and budget boundaries defined
- [x] Startup, per-request, and health-check integration flows diagrammed

### Architecture Readiness

**Status: READY FOR IMPLEMENTATION**

**Key strengths:**
- Zero ambiguity in concurrency model — `sync.RWMutex` with `defer` discipline eliminates race conditions
- Single entry point (`LoadProfiles`) means all validation errors surface at startup, not runtime
- Sentinel errors are distinct and non-overlapping — gateway HTTP mapping is unambiguous
- Config-driven provider/profile management — FR22 and FR23 require no code changes for extension

**Deferred to growth phase (not MVP):**
- Per-caller rate guardrails (in-process sliding window)
- Budget persistence across restarts
- Premium/third-provider profile
