---
stepsCompleted: [step-01-validate-prerequisites, step-02-design-epics, step-03-create-stories, step-04-final-validation]
inputDocuments:
  - docs/terminus/inference/provider-routing/prd.md
  - docs/terminus/inference/provider-routing/architecture.md
initiative: terminus-inference-provider-routing
---

# terminus-inference-provider-routing — Epic Breakdown

## Overview

Complete epic and story breakdown for the provider routing library (`internal/routing/` inside `terminus-inference-gateway`), decomposing the 23 FRs, 10 NFRs, and architecture decisions into implementable stories.

## Requirements Inventory

### Functional Requirements

FR1: The gateway can resolve an inference request to a provider name by passing workload class to the router — without any routing logic in gateway handlers.
FR2: The router selects a provider by evaluating the provider priority chain configured for the request's workload class.
FR3: The router supports at minimum two workload classes: `interactive` (cloud-first) and `batch` (local-first).
FR4: Each workload class has an independently configurable provider priority chain.
FR5: When the first-priority provider is budget-exhausted or unavailable, the router selects the next eligible provider in the chain.
FR6: When fallback occurs, the router surfaces `degraded: true`, `fallback_provider`, and `degradation_reason` in the response context for the gateway to propagate.
FR7: When no eligible provider exists in the chain, the router returns a `no_eligible_provider` error — no silent routing.
FR8: The operator can declare a daily dollar budget per provider in the routing config.
FR9: The router converts the dollar budget to a token limit at startup using the declared `price_per_1k_tokens`.
FR10: The gateway can record actual token spend after each provider response by calling `RecordUsage` with prompt and completion token counts.
FR11: The router tracks cumulative token spend per provider in-process and marks a provider as budget-exhausted when the token limit is reached.
FR12: A provider with `daily_budget_usd: 0` is treated as unlimited — no budget tracking performed.
FR13: Budget state resets on process restart. Persistence across restarts is not required at MVP.
FR14: The operator can define all routing policy in a single YAML file with a declared `schema_version`.
FR15: Provider API keys are referenced by environment variable name in config — no plaintext credentials in YAML.
FR16: The routing layer validates the full config at startup and exits with a field-level error message if any required field is missing or invalid.
FR17: A successful config load is confirmed by a structured startup log line.
FR18: The router logs the selected provider, workload class, and whether degradation occurred for every resolved request.
FR19: Routing logs never include prompt content, response content, or credential values.
FR20: The router exposes a `HealthCheck` method returning per-provider budget state for inclusion in the gateway's `/readyz` response.
FR21: The gateway's `/readyz` reports the routing layer as unhealthy if no providers are eligible for any configured workload class.
FR22: A new provider can be added to the routing layer by adding a provider entry and updating profile priority chains in config — no code changes required.
FR23: A new workload class can be added by adding a new profile entry in config — no code changes required.

### Non-Functional Requirements

NFR-P1: Route resolution (`Resolve`) completes in < 1ms p99 — map lookup plus counter check, no network call.
NFR-P2: `RecordUsage` is non-blocking — in-process counter update only, no I/O.
NFR-P3: `HealthCheck` completes in < 5ms — reads in-memory state only.
NFR-S1: No API key or credential values appear in any log output, error message, or `HealthStatus` response.
NFR-S2: No prompt content, response content, or caller identity tokens appear in routing logs.
NFR-S3: API keys resolved from environment variables at startup only — never read from config at runtime.
NFR-S4: The routing layer does not perform its own authentication — caller identity from gateway auth middleware context.
NFR-I1: The `Router` interface is the sole integration boundary — no direct package dependencies beyond interface and types.
NFR-I2: Compile cleanly against the Go version used by the gateway — no separate Go version requirements.
NFR-I3: No transitive dependencies that conflict with existing gateway dependencies.
NFR-R1: A panic in the routing layer must not crash the gateway — recover internally and return `ErrNoEligibleProvider`.
NFR-R2: A missing or unrecognized workload class returns `ErrInvalidWorkloadClass` cleanly — never panics.
NFR-R3: If all providers become budget-exhausted, return `ErrNoEligibleProvider` — no silent routing, no blocking.

### Additional Requirements (from Architecture)

- Package: `internal/routing/` inside `terminus-inference-gateway` — no new module or repo
- Files: `router.go`, `budget.go`, `config.go`, `errors.go`, `types.go`, `*_test.go` (co-located)
- No framework — stdlib only (`sync`, `errors`, `log/slog`, `os`) + `gopkg.in/yaml.v3`
- `sync.RWMutex` per provider: RLock for Resolve, Lock for RecordUsage
- `ROUTING_CONFIG_PATH` env var is only config path injection point; empty = hard error
- Three error sentinels: `ErrNoEligibleProvider` (→ 503), `ErrBudgetExhausted`, `ErrInvalidWorkloadClass` (→ 400)
- `Resolve` returns `(*RoutingResponse, error)` — carries Provider name + degradation fields
- `HealthStatus` + `ProviderStatus` struct shapes defined (budget_used_usd, budget_limit_usd, budget_exhausted)
- `DegradationReasonBudgetExhausted` and `DegradationReasonUnhealthy` string constants
- `SupportedSchemaVersion = 1`; schema_version required in YAML; unsupported version = hard error
- Config zero-value: `daily_budget_usd: 0` = unlimited; negative = validation error
- Budget reset is process-start-relative (no clock-based reset)
- Token fallback when usage nil: `len(requestBodyJSON) / 4`; log `slog.Warn`
- Panic recovery in `Resolve` via named-return defer/recover
- `testdata/` subdirectory allowed for `config_test.go` YAML fixtures
- All tests must run with `-race`; CI gate: `go test -race ./internal/routing/...`
- Logging allow-list: provider name, workload class, token counts, degradation, budget transitions, config path
- API key env-var names resolved by `LoadProfiles`; resolved keys never in any logged or exported type
- Error wrapping convention: `fmt.Errorf("routing: {action}: %w", err)`
- HTTP error mapping additions to `internal/errors`: `ErrNoEligibleProvider→503`, `ErrInvalidWorkloadClass→400`
- Caller identity NOT in `RoutingRequest` — in context, not used for routing in MVP

### FR Coverage Map

FR1: Epic 2 — Resolve accepts workload class; no routing logic in gateway handlers
FR2: Epic 2 — Priority chain evaluation per workload class
FR3: Epic 2 — `interactive` + `batch` workload classes
FR4: Epic 2 — Independent chains per profile
FR5: Epic 3 — Skip exhausted provider, select next eligible
FR6: Epic 3 — `degraded`, `fallback_provider`, `degradation_reason` in RoutingResponse
FR7: Epic 2 — `ErrNoEligibleProvider` when all providers unavailable
FR8: Epic 3 — `daily_budget_usd` per provider in config
FR9: Epic 3 — Dollar→token conversion at LoadProfiles startup
FR10: Epic 3 — `RecordUsage` with actual prompt+completion counts
FR11: Epic 3 — Cumulative per-provider tracking; exhaustion threshold
FR12: Epic 3 — `daily_budget_usd: 0` = unlimited, no tracking
FR13: Epic 3 — Budget resets on process restart (process-start-relative)
FR14: Epic 1 — YAML config with `schema_version`
FR15: Epic 1 — API keys as env var names in config; resolved at startup
FR16: Epic 1 — Field-level validation; hard exit on startup errors
FR17: Epic 1 — Structured startup log on successful load
FR18: Epic 4 — Log provider, workload class, degradation per request
FR19: Epic 4 — Never log prompt/response/credentials
FR20: Epic 4 — `HealthCheck` returns per-provider budget state
FR21: Epic 4 + Epic 5 — `/readyz` unhealthy if all providers exhausted (wired in Epic 5)
FR22: Epic 2 — Add provider via config only; no code change
FR23: Epic 2 — Add workload class via config only; no code change

NFR-P1–P3: Cross-cutting — addressed in Epic 2 (Resolve) and Epic 3 (RecordUsage) stories
NFR-S1–S4: Cross-cutting — addressed in Epics 1, 2, 4 stories (logging allow-list, env var resolution)
NFR-I1–I3: Cross-cutting — addressed in Epic 1 (package scaffold, no dep conflicts)
NFR-R1–R3: Cross-cutting — addressed in Epic 2 (panic recovery, clean error returns) and Epic 3

## Epic List

### Epic 1: Operator can configure routing policy in YAML

The routing layer loads a validated YAML config at startup, resolves API keys from environment variables, and exits with field-level error messages if anything is invalid. This is the foundation all other epics build on.

**FRs covered:** FR14, FR15, FR16, FR17
**NFRs:** NFR-I1 (Router interface boundary), NFR-I2 (same Go version), NFR-I3 (no dep conflicts), NFR-S3 (env-only key resolution)
**Architecture:** Package scaffold (all 5 source files + test files), `ROUTING_CONFIG_PATH` env var, `SupportedSchemaVersion = 1`, `LoadProfiles`, canonical YAML schema, all sentinel errors defined

### Epic 2: Gateway can route requests to the correct provider

`Resolve` evaluates the provider priority chain for the requested workload class and returns the highest-priority available provider. Unrecognized workload classes and all-exhausted states return clean sentinel errors. New providers and profiles are added via config only.

**FRs covered:** FR1, FR2, FR3, FR4, FR7, FR22, FR23
**NFRs:** NFR-P1 (<1ms resolve), NFR-R1 (panic recovery in Resolve), NFR-R2 (ErrInvalidWorkloadClass), NFR-R3 (no silent routing), NFR-S2 (no PII in logs)
**Architecture:** `Router` interface, `RoutingRequest`/`RoutingResponse` types, `ErrNoEligibleProvider`, `ErrInvalidWorkloadClass`, panic recovery, slog allow-list, HTTP error mapping in `internal/errors`

### Epic 3: Batch workloads can't exhaust cloud credits

`RecordUsage` accumulates actual token spend per provider. When a provider crosses its budget limit it is marked exhausted and `Resolve` skips it automatically, falling back to the next eligible provider with degradation fields populated in the response.

**FRs covered:** FR5, FR6, FR8, FR9, FR10, FR11, FR12, FR13
**NFRs:** NFR-P2 (RecordUsage non-blocking), NFR-R3 (no silent routing when exhausted)
**Architecture:** `BudgetTracker`, `sync.RWMutex` (RLock/Lock discipline), dollar→token conversion, zero-value semantics, token fallback formula (`len/4`), `DegradationReason` constants, budget reset on restart

### Epic 4: Operator can observe routing health

Every routing decision is logged with provider name, workload class, and degradation status. `HealthCheck` exposes per-provider budget state. No credentials, prompt content, or response content ever appears in logs or health output.

**FRs covered:** FR18, FR19, FR20
**NFRs:** NFR-S1 (no keys in logs/HealthStatus), NFR-S2 (no prompt/response in logs), NFR-P3 (HealthCheck <5ms)
**Architecture:** `HealthStatus` + `ProviderStatus` struct shapes, slog never-log enforcement, `HealthCheck` implementation

### Epic 5: Gateway can deploy with routing enabled

The gateway starts up with the routing layer active. The HTTP handler calls `Resolve` and `RecordUsage` in the request pipeline. `HealthCheck` is wired into the existing `/readyz` endpoint. An operator can deploy by setting `ROUTING_CONFIG_PATH` and providing a valid config file — no code changes required.

**FRs covered:** FR1 (gateway-side wiring), FR21 (readyz integration)
**Architecture:** `main.go` wiring (`LoadProfiles` → inject `Router`), HTTP handler integration, `/readyz` wiring, `ROUTING_CONFIG_PATH` in Helm/ArgoCD config, sample `routing-config.yaml` deployment artifact, smoke test

---

## Epic 1: Operator can configure routing policy in YAML

The routing layer loads a validated YAML config at startup, resolves API keys from environment variables, and exits with field-level error messages if anything is invalid. All downstream epics build on this scaffold.

### Story 1.1: Package scaffold and sentinel errors

As a **gateway developer**,
I want the `internal/routing/` package to exist with the full file scaffold and exported sentinel errors,
So that all subsequent routing stories have a valid Go package to build into.

**Acceptance Criteria:**

**Given** the gateway module exists
**When** `internal/routing/` is created with `router.go`, `budget.go`, `config.go`, `errors.go`, `types.go`
**Then** `go build ./internal/routing/...` succeeds with no errors
**And** `errors.go` exports `ErrNoEligibleProvider`, `ErrBudgetExhausted`, `ErrInvalidWorkloadClass`
**And** `config.go` exports `const SupportedSchemaVersion = 1`
**And** `router.go` declares the `Router` interface with `Resolve`, `RecordUsage`, `HealthCheck` signatures matching the architecture specification
**And** the package compiles on the same Go version as the gateway with zero new external dependencies (only `gopkg.in/yaml.v3`)

---

### Story 1.2: LoadProfiles parses and validates the YAML config

As an **operator**,
I want `LoadProfiles` to parse a YAML routing config and return a validation error if anything is wrong,
So that a misconfigured gateway fails loudly at startup rather than silently misbehaving.

**Acceptance Criteria:**

**Given** `ROUTING_CONFIG_PATH` points to a valid YAML file matching the canonical schema with `schema_version: 1`
**When** `LoadProfiles(os.Getenv("ROUTING_CONFIG_PATH"))` is called
**Then** it returns a `(Router, nil)` with all providers and profiles loaded into unexported runtime state
**And** an `slog.Info` log line is emitted confirming load success, containing only the config file path (no content, no keys)

**Given** `ROUTING_CONFIG_PATH` is empty or unset
**When** `LoadProfiles("")` is called
**Then** it returns `(nil, error)` with message `"routing: ROUTING_CONFIG_PATH environment variable is not set"`

**Given** the YAML file declares `schema_version: 2`
**When** `LoadProfiles` is called
**Then** it returns `(nil, error)` naming the unsupported version and the supported value (`1`)

**Given** a required field (e.g. `price_per_1k_tokens`) is missing from a provider entry
**When** `LoadProfiles` is called
**Then** it returns `(nil, error)` naming the exact field and provider name in the message

**Given** `daily_budget_usd` is set to a negative value for any provider
**When** `LoadProfiles` is called
**Then** it returns `(nil, error)`; negative budget values are rejected

**Given** `profiles` references a provider name not declared in the `providers` list
**When** `LoadProfiles` is called
**Then** it returns `(nil, error)` naming the undefined provider reference

---

### Story 1.3: LoadProfiles resolves API keys from environment variables

As an **operator**,
I want provider API keys to be referenced by env var name in config and resolved at startup,
So that no plaintext credentials ever appear in the YAML file or leak through the system.

**Acceptance Criteria:**

**Given** a provider entry has `api_key_env: OPENAI_API_KEY` and the env var is set to a non-empty value
**When** `LoadProfiles` is called
**Then** the resolved key is stored in an unexported runtime field on the provider's internal entry
**And** the resolved key value is never present in any exported type, log output, error message, or `HealthStatus`

**Given** `api_key_env: OPENAI_API_KEY` and the env var is NOT set (empty or missing)
**When** `LoadProfiles` is called
**Then** it returns `(nil, error)` with a message that names the provider and the missing env var name
**And** the error message does NOT contain any credential value

**Given** `api_key_env: ""` (empty string — e.g. a local unauthenticated provider)
**When** `LoadProfiles` is called
**Then** no env var lookup is attempted and the provider loads successfully with an empty key

---

### Story 1.4: Config tests with testdata fixtures

As a **developer**,
I want `config_test.go` to cover all `LoadProfiles` paths using YAML fixture files run with the race detector,
So that config validation behaviour is regression-tested and concurrent-access safe.

**Acceptance Criteria:**

**Given** `internal/routing/testdata/` contains fixture YAML files for: valid config, unsupported schema version, missing required field, missing env var, negative budget, and undefined provider reference
**When** `go test -race ./internal/routing/...` is run
**Then** all table-driven cases in `config_test.go` pass with zero race detector findings
**And** each fixture file represents exactly one scenario (one file per case)
**And** no fixture file contains a real API key value — test env vars are set inline within the test

---

## Epic 2: Gateway can route requests to the correct provider

`Resolve` evaluates the provider priority chain for the requested workload class and returns the highest-priority available provider. Unrecognized workload classes and all-exhausted states return clean sentinel errors. Fallback, new providers, and new workload classes are config-only operations.

### Story 2.1: RoutingRequest, RoutingResponse, and shared types

As a **gateway developer**,
I want the shared request/response types for the routing package defined in `types.go`,
So that `Resolve` and all callers have a single canonical type definition to import.

**Acceptance Criteria:**

**Given** `types.go` is implemented
**Then** `RoutingRequest` contains at minimum: `Profile string` (workload class name)
**And** `RoutingResponse` contains: `Provider string`, `Degraded bool`, `FallbackProvider string`, `DegradationReason string`
**And** `HealthStatus` contains: `Providers []ProviderStatus`, `Degraded bool`, `AllExhausted bool`
**And** `ProviderStatus` contains: `Name string`, `BudgetLimitUSD float64`, `BudgetUsedUSD float64`, `BudgetExhausted bool`
**And** `DegradationReasonBudgetExhausted` and `DegradationReasonUnhealthy` string constants are declared
**And** all types compile cleanly with no circular imports

---

### Story 2.2: Resolve selects the highest-priority eligible provider

As a **gateway operator**,
I want `Resolve` to evaluate the provider priority chain for the requested workload class and return the first eligible provider,
So that inference requests are dispatched to the correct provider without any routing logic in HTTP handlers.

**Acceptance Criteria:**

**Given** a `RoutingRequest` with `Profile: "interactive"` and the `interactive` profile has priority chain `[openai, ollama]`
**When** `Resolve` is called and `openai` is eligible
**Then** it returns `(&RoutingResponse{Provider: "openai", Degraded: false}, nil)`

**Given** a `RoutingRequest` with `Profile: "batch"` and the `batch` profile has priority chain `[ollama, openai]`
**When** `Resolve` is called and `ollama` is eligible
**Then** it returns `(&RoutingResponse{Provider: "ollama", Degraded: false}, nil)`

**Given** a profile with a three-provider chain `[a, b, c]` and provider `a` is unavailable
**When** `Resolve` is called
**Then** it returns `(&RoutingResponse{Provider: "b", Degraded: false}, nil)` (skips `a`, selects `b`)

**Given** `Resolve` is called concurrently by 100 goroutines with the same profile
**When** `go test -race` is run
**Then** zero race conditions are detected

---

### Story 2.3: Resolve returns clean sentinel errors for invalid and exhausted states

As a **gateway developer**,
I want `Resolve` to return typed sentinel errors for unrecognized workload classes and when no eligible provider exists,
So that callers can map these to correct HTTP status codes without inspecting error strings.

**Acceptance Criteria:**

**Given** a `RoutingRequest` with `Profile: "unknown-class"` not defined in any loaded profile
**When** `Resolve` is called
**Then** it returns `(nil, err)` where `errors.Is(err, ErrInvalidWorkloadClass)` is true
**And** the error message includes the unrecognized profile name

**Given** all providers in the chain for the requested profile are unavailable
**When** `Resolve` is called
**Then** it returns `(nil, err)` where `errors.Is(err, ErrNoEligibleProvider)` is true
**And** no panic occurs under any input

**Given** a panic is induced inside `Resolve` (e.g. via test injection)
**When** the panic fires
**Then** the deferred recover catches it, logs an `slog.Error` with the recovered value, and returns `(nil, ErrNoEligibleProvider)`
**And** the gateway process does not crash

---

### Story 2.4: HTTP error mapping for routing sentinel errors

As a **gateway developer**,
I want `internal/errors` to map routing sentinel errors to the correct HTTP status codes,
So that callers receive 503 for no-eligible-provider and 400 for invalid-workload-class without custom handler logic.

**Acceptance Criteria:**

**Given** an error where `errors.Is(err, routing.ErrNoEligibleProvider)` is true
**When** it reaches the gateway's error mapping layer in `internal/errors`
**Then** it maps to `HTTP 503 Service Unavailable`

**Given** an error where `errors.Is(err, routing.ErrInvalidWorkloadClass)` is true
**When** it reaches the gateway's error mapping layer
**Then** it maps to `HTTP 400 Bad Request`

**Given** a wrapped error `fmt.Errorf("routing: no eligible provider for profile %q: %w", profile, ErrNoEligibleProvider)`
**When** `errors.Is(err, routing.ErrNoEligibleProvider)` is evaluated
**Then** it returns `true` (wrapping must not break sentinel matching)

---

### Story 2.5: router_test.go — table-driven Resolve tests

As a **developer**,
I want `router_test.go` to cover all `Resolve` paths with table-driven tests and the race detector,
So that routing logic is regression-safe and concurrent access is validated.

**Acceptance Criteria:**

**Given** `router_test.go` uses table-driven tests in `package routing` (white-box)
**When** `go test -race ./internal/routing/...` is run
**Then** all cases pass: happy path (first provider), fallback (second provider), invalid profile, all-unavailable, panic recovery
**And** mock providers implement the gateway `Provider` interface — no test-only interfaces are created
**And** all test cases use deterministic, non-network inputs

---

## Epic 3: Batch workloads can't exhaust cloud credits

`RecordUsage` accumulates actual token spend per provider after each inference response. When a provider's cumulative spend crosses its token limit, it is marked exhausted and subsequent `Resolve` calls skip it, falling back with degradation metadata in the response.

### Story 3.1: BudgetTracker — per-provider token limit and RWMutex

As a **gateway developer**,
I want `BudgetTracker` in `budget.go` to hold per-provider token limits and running spend with safe concurrent access,
So that budget reads in `Resolve` and budget writes in `RecordUsage` never race.

**Acceptance Criteria:**

**Given** `LoadProfiles` processes a provider with `daily_budget_usd: 5.00` and `price_per_1k_tokens: 0.002`
**When** the `BudgetTracker` is initialised for that provider
**Then** the computed token limit is `5.00 / 0.002 * 1000 = 2,500,000` tokens
**And** the initial running spend is `0`

**Given** a provider with `daily_budget_usd: 0`
**When** the `BudgetTracker` is initialised
**Then** no token limit is set for that provider — budget tracking is disabled (unlimited)

**Given** `BudgetTracker` is accessed concurrently by `Resolve` (RLock) and `RecordUsage` (Lock)
**When** `go test -race` exercises 100 concurrent readers and 10 concurrent writers
**Then** zero race conditions are detected

---

### Story 3.2: RecordUsage accumulates actual token spend

As a **gateway developer**,
I want `RecordUsage` to accumulate actual prompt+completion token counts from provider responses and mark providers exhausted at their limit,
So that dollar budget enforcement is based on real spend, not estimates.

**Acceptance Criteria:**

**Given** a provider with a token limit of 1000 and current spend of 0
**When** `RecordUsage(ctx, "openai", 400, 300)` is called (700 tokens)
**Then** the provider's running spend increases to 700 and `BudgetExhausted` remains `false`

**Given** the same provider now at 700 tokens of spend
**When** `RecordUsage(ctx, "openai", 200, 150)` is called (350 tokens, total 1050)
**Then** the provider's running spend is 1050, exceeding the 1000 limit
**And** `BudgetExhausted` is set to `true` for that provider

**Given** a provider with `daily_budget_usd: 0` (unlimited)
**When** `RecordUsage` is called with any token counts
**Then** no spend is tracked and `BudgetExhausted` remains `false` always

**Given** a provider response where `usage` is nil (token count unavailable)
**When** `RecordUsage` is called with `promptTokens: 0, completionTokens: 0` and the request body length is known
**Then** the estimated token count `len(requestBodyJSON) / 4` is used
**And** an `slog.Warn` is emitted indicating the estimate was used (no request body content in the log)

**Given** `RecordUsage` is called from the response path
**Then** it completes without any I/O or blocking operations — in-process counter update only

---

### Story 3.3: Resolve skips exhausted providers and surfaces degradation

As an **operator**,
I want `Resolve` to automatically skip budget-exhausted providers and fall back to the next eligible provider with degradation metadata in the response,
So that interactive workloads degrade gracefully rather than failing when the primary provider is out of budget.

**Acceptance Criteria:**

**Given** the `interactive` profile has chain `[openai, ollama]` and `openai` is budget-exhausted
**When** `Resolve` is called with `Profile: "interactive"`
**Then** it returns `(&RoutingResponse{Provider: "ollama", Degraded: true, FallbackProvider: "ollama", DegradationReason: "budget_exhausted"}, nil)`

**Given** all providers in the chain are budget-exhausted
**When** `Resolve` is called
**Then** it returns `(nil, err)` where `errors.Is(err, ErrNoEligibleProvider)` is true
**And** no provider is silently selected beyond the declared chain

**Given** the primary provider is budget-exhausted and `Resolve` selects a fallback
**Then** the `Degraded` field on the response is `true` and `FallbackProvider` names the actual provider used

---

### Story 3.4: budget_test.go — table-driven budget and fallback tests

As a **developer**,
I want `budget_test.go` to cover all `RecordUsage` and budget-exhaustion paths with table-driven tests and the race detector,
So that concurrency correctness and threshold behaviour are verified.

**Acceptance Criteria:**

**Given** `budget_test.go` uses table-driven tests in `package routing` (white-box)
**When** `go test -race ./internal/routing/...` is run
**Then** test cases cover: token accumulation below threshold, token accumulation crossing threshold, unlimited provider, nil-usage fallback estimate, concurrent writes from 50 goroutines
**And** zero race conditions are detected in all cases

---

## Epic 4: Operator can observe routing health

Every `Resolve` call emits a structured log line. `HealthCheck` exposes per-provider budget state for gateway health reporting. No credentials, prompt content, or response content ever appears in logs or health output.

### Story 4.1: Resolve emits structured log per request

As an **operator**,
I want every `Resolve` call to emit a structured `slog.Debug` log line with provider, workload class, and degradation status,
So that I can observe routing decisions in production logs without any PII leakage.

**Acceptance Criteria:**

**Given** `Resolve` successfully returns a provider
**When** the request is processed
**Then** an `slog.DebugContext` line is emitted containing: selected provider name, requested profile name, `degraded: false`
**And** the log line does NOT contain prompt text, response text, API key values, or caller identity tokens

**Given** `Resolve` selects a fallback provider due to exhaustion
**When** the request is processed
**Then** an `slog.DebugContext` line is emitted containing: fallback provider name, profile name, `degraded: true`, `degradation_reason`
**And** the log line contains no PII or credential values

**Given** `Resolve` returns `ErrNoEligibleProvider`
**When** the request is processed
**Then** an `slog.WarnContext` line is emitted naming the profile and that no provider was available
**And** no sensitive values appear in the log

---

### Story 4.2: HealthCheck returns per-provider budget state

As an **operator**,
I want `HealthCheck` to return a `HealthStatus` with per-provider budget metrics,
So that the gateway's `/readyz` endpoint can report whether the routing layer is healthy.

**Acceptance Criteria:**

**Given** two providers loaded — `openai` (budget limit 5.00 USD, 2.30 USD used) and `ollama` (unlimited)
**When** `HealthCheck(ctx)` is called
**Then** it returns a `HealthStatus` with `Providers` containing one entry per provider
**And** the `openai` `ProviderStatus` has `BudgetLimitUSD: 5.00`, `BudgetUsedUSD: 2.30`, `BudgetExhausted: false`
**And** the `ollama` `ProviderStatus` has `BudgetLimitUSD: 0`, `BudgetUsedUSD: 0`, `BudgetExhausted: false`
**And** `HealthStatus.AllExhausted` is `false`

**Given** all providers are budget-exhausted
**When** `HealthCheck(ctx)` is called
**Then** `HealthStatus.AllExhausted` is `true` and `HealthStatus.Degraded` is `true`

**Given** `HealthCheck` is called from the gateway's `/readyz` handler
**Then** it completes in under 5ms — reads in-memory state only, no network calls

**Given** `HealthCheck` returns a `HealthStatus`
**Then** no API key value or credential appears in any field of `HealthStatus` or any `ProviderStatus`

---

### Story 4.3: HealthCheck and logging tests

As a **developer**,
I want `router_test.go` to include tests for `HealthCheck` and log output verification,
So that health correctness and PII-safety are regression-tested.

**Acceptance Criteria:**

**Given** `router_test.go` tests `HealthCheck`
**When** `go test -race ./internal/routing/...` is run
**Then** test cases cover: healthy state (no exhaustion), partially exhausted (one provider), fully exhausted (all providers)
**And** a test verifies that a log handler capturing `slog` records contains no credential or prompt content strings across all `Resolve` call paths
**And** zero race conditions are detected

---

## Epic 5: Gateway can deploy with routing enabled

The gateway starts up with the routing layer active. HTTP handlers call `Resolve` and `RecordUsage`. `HealthCheck` feeds the existing `/readyz` endpoint. An operator deploys by setting `ROUTING_CONFIG_PATH` and providing a valid config file — no further code changes required.

### Story 5.1: main.go wires LoadProfiles at startup

As a **gateway developer**,
I want `main.go` to call `LoadProfiles` at startup and inject the returned `Router` into the server,
So that the routing layer is active for every request from process start.

**Acceptance Criteria:**

**Given** `ROUTING_CONFIG_PATH` is set and points to a valid config file
**When** the gateway starts
**Then** `routing.LoadProfiles(os.Getenv("ROUTING_CONFIG_PATH"))` is called before the HTTP server begins accepting requests
**And** if `LoadProfiles` returns an error, the gateway logs the error and exits with a non-zero code
**And** the returned `Router` is injected into the server struct — no global variable

**Given** `ROUTING_CONFIG_PATH` is not set
**When** the gateway starts
**Then** it exits immediately with `"routing: ROUTING_CONFIG_PATH environment variable is not set"`

---

### Story 5.2: HTTP handler calls Resolve and RecordUsage

As a **gateway developer**,
I want the primary inference HTTP handler to call `Resolve` before dispatching and `RecordUsage` after receiving the provider response,
So that every inference request flows through the routing layer.

**Acceptance Criteria:**

**Given** an inbound inference request with `X-Workload-Class: interactive` (or equivalent routing signal)
**When** the handler processes the request
**Then** `router.Resolve(ctx, &RoutingRequest{Profile: workloadClass})` is called before provider dispatch
**And** if `Resolve` returns `ErrNoEligibleProvider`, the handler returns HTTP 503
**And** if `Resolve` returns `ErrInvalidWorkloadClass`, the handler returns HTTP 400

**Given** the provider returns a successful response with `usage.PromptTokens` and `usage.CompletionTokens`
**When** the handler finishes reading the response
**Then** `router.RecordUsage(ctx, providerName, usage.PromptTokens, usage.CompletionTokens)` is called
**And** `RecordUsage` is called on every successful response — not only when degraded

**Given** the provider response includes degradation info (`RoutingResponse.Degraded == true`)
**When** the handler writes its response to the caller
**Then** the response includes `degraded: true`, `fallback_provider`, and `degradation_reason` fields

---

### Story 5.3: /readyz wires HealthCheck

As an **operator**,
I want the gateway's `/readyz` endpoint to include routing layer health in its response,
So that orchestration infrastructure (k3s, ArgoCD) can detect when all providers are exhausted.

**Acceptance Criteria:**

**Given** the `/readyz` handler exists in the gateway
**When** `router.HealthCheck(ctx)` returns `HealthStatus.AllExhausted: false`
**Then** `/readyz` returns HTTP 200 and the routing section of the response shows healthy

**Given** `router.HealthCheck(ctx)` returns `HealthStatus.AllExhausted: true`
**When** `/readyz` is called
**Then** it returns HTTP 503 — the gateway reports itself unhealthy
**And** the response body includes per-provider budget state from `HealthStatus.Providers`

---

### Story 5.4: Deployment artifacts — config and Helm/ArgoCD

As an **operator**,
I want a sample `routing-config.yaml` committed alongside the deployment manifests and `ROUTING_CONFIG_PATH` added to the Helm chart,
So that deploying the routing-enabled gateway requires only setting an env var and providing the config file.

**Acceptance Criteria:**

**Given** the gateway Helm chart / ArgoCD application config
**When** deployment manifests are reviewed
**Then** `ROUTING_CONFIG_PATH` is declared as an environment variable in the chart (value configurable per environment)
**And** a `routing-config.yaml` example file is committed to the gateway deployment directory with both `interactive` and `batch` profiles, two providers (ollama + openai stubs), and valid `schema_version: 1`
**And** the example file contains no real API key values — only `api_key_env` references to env var names

---

### Story 5.5: Smoke test — routing layer active in deployed gateway

As an **operator**,
I want a smoke test confirming the deployed gateway routes inference requests through the routing layer,
So that a broken deployment is detected before it reaches production.

**Acceptance Criteria:**

**Given** the gateway is deployed with a valid `routing-config.yaml` and `ROUTING_CONFIG_PATH` set
**When** a test inference request is sent with `Profile: "interactive"`
**Then** the response includes a selected provider name and HTTP 200
**And** `/readyz` returns 200 with routing health populated

**Given** a test inference request is sent with `Profile: "unknown"`
**When** the handler processes it
**Then** the response is HTTP 400

**Given** no provider is reachable (simulated by setting budgets to near-zero in the test config and exhausting them)
**When** a test inference request is sent
**Then** the response is HTTP 503 and `/readyz` reports unhealthy
