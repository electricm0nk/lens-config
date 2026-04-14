---
stepsCompleted: [step-01-init, step-02-context, step-03-starter, step-04-decisions, step-05-patterns, step-06-structure, step-07-validation, step-08-complete]
status: complete
completedAt: 2026-04-03
inputDocuments:
  - docs/terminus/inference/gateway/product-brief.md
  - docs/terminus/inference/gateway/prd.md
workflowType: architecture
project_name: terminus-inference-gateway
user_name: electricm0nk
date: 2026-04-03
---

# Architecture Decision Document

_This document builds collaboratively through step-by-step discovery. Sections are appended as we work through each architectural decision together._

## Project Context Analysis

### Requirements Overview

**Functional Requirements (23 total):**
Organized into six groups: request handling, provider abstraction, authentication & authorization, error contract, observability & health, and contract/interoperability. The center of gravity is provider abstraction — FR5 through FR8 define the adapter interface shape and registration model, which is the core deliverable of this feature.

**Non-Functional Requirements driving architecture:**
- **NFR-P1:** 100ms gateway overhead budget per request — constrains middleware choices
- **NFR-P2:** Configurable provider timeout, conservatively defaulting to 5 minutes for batch
- **NFR-S1–S4:** TLS-only inter-service, credentials never logged or forwarded, no session caching, auth on every request
- **NFR-R3:** Per-adapter failure isolation — one adapter failing must not affect gateway availability for others
- **NFR-I1:** terminus-infra-secrets dependency is a startup gate, not a runtime fallback

**Scale & Complexity:**

- Primary domain: API backend — internal platform service (greenfield on brownfield)
- Complexity level: high (abstraction correctness, not feature volume)
- Estimated architectural components: ~6 (gateway core, adapter interface, Ollama adapter, auth middleware, health subsystem, config loader)

### Technical Constraints & Dependencies

- **k3s deployment substrate** (terminus-infra-k3s) — deployment target is a known k3s workload environment; Helm chart assumed
- **Vault / k8s service account auth** (terminus-infra-secrets) — identity is delegated; gateway owns enforcement, not issuance
- **OpenAI API v1 schema locked** — versioning constraint is non-negotiable; incompatibility is treated as a breaking change requiring a new initiative
- **No provider-specific routing policy in gateway handlers** — hard boundary from service plan; routing belongs to terminus-inference-provider-routing

### Cross-Cutting Concerns Identified

- **Auth enforcement** — must intercept every request before provider resolution; no bypass paths
- **Error normalization** — adapters return raw provider responses; gateway pipeline must normalize before returning to caller
- **Provider health propagation** — per-adapter health state surfaced on `/readyz`; failure isolation required
- **Config-driven provider registration** — provider list is config, not code; adapter registration must not require handler changes
- **Observability** — health/readiness endpoints are first-class; no silent failure modes permitted

## Starter Template Evaluation

### Primary Technology Domain

API backend — Go, based on architectural fit (gateway/proxy workload) and organizational trajectory.
TypeScript was initially considered for platform alignment but rejected: the gateway shares zero code with terminus.platform, so there is no alignment benefit. C# was considered and rejected due to ecosystem isolation on a k3s/Vault/Go-skewing platform.

### Starter Options Considered

- **TypeScript (Fastify)** — rejected: platform alignment argument doesn't hold; gateway shares no code with terminus.platform
- **C# (.NET 8 minimal API)** — rejected: ecosystem isolation on a platform that skews Go; heavier container footprint
- **Go** — selected

### Selected Starter: Go standard library + `chi` router

**Rationale for Selection:**
Go's `net/http` + [`chi`](https://github.com/go-chi/chi) provides a minimal, idiomatic HTTP router with middleware composability. The adapter pattern maps directly onto Go interfaces — compile-time enforced, no framework overhead. `oapi-codegen` generates server stubs and types from the OpenAPI spec, making FR21–FR23 (spec as source-of-truth) a first-class build step. Single static binary deployment on k3s. Satisfies NFR-P1 (100ms overhead) with significant headroom.

**Initialization:**

```bash
mkdir terminus-inference-gateway && cd terminus-inference-gateway
go mod init github.com/electricm0nk/terminus-inference-gateway
go get github.com/go-chi/chi/v5
go get github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen@latest
```

**Architectural Decisions Provided by Starter:**

**Language & Runtime:** Go (latest stable), single static binary

**HTTP Router:** `chi` — lightweight, idiomatic, middleware-chainable; maps cleanly to OpenAI-compatible route surface

**Schema & API Spec:** `oapi-codegen` — generates Go types and server interface stubs from OpenAPI 3.0 spec; spec is source-of-truth for contract tests (FR21–FR23)

**Adapter Interface:** Go interface — compile-time enforcement; each provider is a struct implementing the interface

**Testing:** Go standard `testing` + `net/http/httptest` for handler tests; contract tests driven from generated types

**Build Tooling:** `go build` → single static binary; `FROM scratch` or `FROM gcr.io/distroless/static` container image

**Code Organization:** flat package structure for MVP (`/cmd/gateway`, `/internal/adapter`, `/internal/auth`, `/internal/provider`, `/api/`)

**Note:** Project initialization using this scaffold should be the first implementation story. `oapi-codegen` configuration file should be committed alongside the OpenAPI spec.

## Core Architectural Decisions

### Decision Priority Analysis

**Critical Decisions (Block Implementation):**
- Auth mechanism: Vault token lookup on every request
- Error handling: Always normalize at gateway boundary, never leak adapter internals
- Config: Environment variables for provider registration (k8s-native)
- Request lifecycle: Per-request context deadline from configurable timeout

**Important Decisions (Shape Architecture):**
- Container image: `distroless/static` (CA certificates available, no shell)
- Helm chart: Co-located in gateway repo, not in terminus-infra
- Health endpoints: Manual handlers, no framework dependency

**Deferred Decisions (Post-MVP):**
- Service mesh auth delegation (Istio/Linkerd mTLS) — clean upgrade path once mesh is in place
- Rate limiting — deferred to terminus-inference-provider-routing (MVP2)
- k8s service account TokenReview — can replace Vault lookup when service mesh matures
- Streaming (`stream: true`) — out of scope for MVP; gateway returns 501 Not Implemented for streaming requests
- Observability/telemetry (metrics, tracing, token count logging) — deferred to a dedicated logging/observability initiative. TODO: domain constitution Art.10 requires route id, duration, token counts, outcome, and cost tag per request — must be addressed before production readiness.

### Data Architecture

No database. Gateway is stateless — no persistent data owned at this layer.

**Config loading:** Environment variables (k8s ConfigMap / Secret injection)
- Provider registration: env vars per adapter (base URL, auth env var reference)
- Timeout configuration: env var with conservative default (5 min for batch per NFR-P2)
- Vault address: env var, required at startup

### Authentication & Security

**Mechanism:** `Authorization: Bearer <vault-token>` on every inbound request
**Validation:** Vault `/v1/auth/token/lookup-self` call per request — no session caching (NFR-S4)
**Startup gate:** Vault client must reach Vault at startup; gateway refuses to start if unreachable (NFR-I1)
**Credentials:** Never logged, never forwarded to provider adapters, never included in error responses (NFR-S2)
**TLS:** All inter-service communication over TLS (NFR-S1); Vault communication uses TLS by default
**Token lifecycle (MVP):** Non-expiring periodic token stored in a k8s Secret; injected into the gateway container via `secretKeyRef` in the Helm deployment template. TODO: evaluate migration to k8s auth method (service account → dynamic short-lived token) post-MVP.
**Token injection (k8s):** `kubectl create secret generic gateway-vault-token --from-literal=GATEWAY_VAULT_TOKEN=<token>`. Helm `vault.tokenSecret` value references this Secret by name.

### API & Communication Patterns

**API surface:** OpenAI API v1 schema (locked — non-negotiable constraint)
**Error normalization:** All adapter errors mapped to normalized error envelope at gateway boundary; provider internals never reach caller (NFR-S3)
**Request timeout:** `context.WithTimeout` per request using configurable timeout value (NFR-P2); gateway deadline is independent of inbound connection deadline
**Spec generation:** `oapi-codegen` from colocated OpenAPI 3.0 spec; spec is source-of-truth for contract tests (FR21–FR23)

### Infrastructure & Deployment

**Container base image:** `gcr.io/distroless/static` — CA certificates available for Vault TLS; no shell attack surface
**Helm chart:** Co-located in gateway repository (`/deploy/helm/`); not in terminus-infra
**Health endpoints:** Manual handlers — `/healthz` (liveness, returns 200), `/readyz` (per-adapter health check)
**Build:** `go build -ldflags="-s -w"` → single static binary; multi-stage Dockerfile

**Helm values.yaml sketch:**
```yaml
replicaCount: 1

image:
  repository: ghcr.io/electricm0nk/terminus-inference-gateway
  tag: latest
  pullPolicy: IfNotPresent

vault:
  addr: ""        # required — e.g. https://vault.trantor.internal:8200
  tokenSecret: "" # required — name of k8s Secret containing GATEWAY_VAULT_TOKEN

providers:
  ollama:
    enabled: true
    baseURL: ""     # required when enabled — e.g. http://ollama.trantor.internal
    timeoutSeconds: 300

gateway:
  port: 8080
  logLevel: info

service:
  type: ClusterIP
  port: 80

resources: {}
```

**oapi-codegen.yaml config:**
```yaml
package: gen
generate:
  strict-server: true   # generates StrictServerInterface — compile-time request/response types
  models: true
  embedded-spec: false
output: api/gen/gen.go
output-options:
  skip-prune: false
```

### Decision Impact Analysis

**Implementation Sequence:**
1. OpenAPI spec + `oapi-codegen` scaffold (generates types and server interface stubs)
2. Provider adapter interface definition + Ollama adapter
3. Auth middleware (Vault token validation)
4. Request pipeline (auth → adapter resolution → forward → normalize → return)
5. Health/readiness handlers
6. Config loader (env vars)
7. Helm chart + Dockerfile
8. Contract test suite against generated types

**Cross-Component Dependencies:**
- Auth middleware depends on Vault client; Vault client configured from env vars
- Provider adapters implement the adapter interface; registered via config at startup
- Error normalization happens in the request pipeline before response is written, not in adapter layer
- `/readyz` endpoint calls each registered adapter's health check method — adapter interface must include a `Health() error` method

## Implementation Patterns & Consistency Rules

### Pattern Categories Defined

**Critical Conflict Points Identified:** 7 areas where AI agents could make different choices

### Naming Patterns

**Package Naming:**
- All package names: lowercase, single word (`adapter`, `auth`, `config`, `provider`, `health`)
- No underscores in package names
- Internal-only packages under `internal/`; no `pkg/` for MVP

**Exported Symbol Naming:**
- Interfaces: noun describing the role, no `I` prefix (`Provider`, `Adapter`, not `IProvider`)
- Struct types: PascalCase (`OllamaAdapter`, `GatewayConfig`, `ErrorEnvelope`)
- Error variables: `Err` prefix (`ErrProviderUnavailable`, `ErrUnauthorized`)
- Constructor functions: `New` prefix (`NewOllamaAdapter`, `NewGatewayConfig`)

**Environment Variable Naming:**
- All caps, underscore-separated, `GATEWAY_` prefix for gateway config
- Provider-specific: `GATEWAY_OLLAMA_BASE_URL`, `GATEWAY_OLLAMA_TIMEOUT_SECONDS`
- Vault: `GATEWAY_VAULT_ADDR`, `GATEWAY_VAULT_TOKEN` (for local dev only — k8s injects)

**Logging Field Names (structured logging, `log/slog`):**
- `provider` — provider adapter name
- `route` — request route/path
- `request_id` — per-request trace ID
- `status` — HTTP status code
- `error` — error string (never the raw error object)
- Never log credential values, authorization headers, or provider response bodies

### Structure Patterns

**Package Organization:**
```
/cmd/gateway/          — main entry point only; no business logic
/internal/adapter/     — Provider interface + all adapter implementations
/internal/auth/        — Vault token validation middleware
/internal/config/      — Config struct + env var loader
/internal/health/      — /healthz and /readyz handlers
/internal/pipeline/    — Request pipeline (auth → resolve → forward → normalize)
/internal/errors/      — Error envelope types and mapping functions
/api/                  — OpenAPI spec (.yaml) + oapi-codegen output (do not edit generated files)
/deploy/helm/          — Helm chart
```

**Test File Placement:**
- Co-located with the package under test: `internal/adapter/ollama_test.go`
- All tests: table-driven (`tt := []struct{...}`) — no single-case test functions for happy/sad path splits
- Integration tests: `_integration_test.go` suffix, skipped unless `INTEGRATION=true` env var set

### Format Patterns

**API Response Formats:**
- Success: direct OpenAI-compatible response body — no wrapper envelope
- Errors: always use the error envelope:
  ```json
  { "error": { "type": "...", "provider": "...", "reason": "...", "message": "..." } }
  ```
- `provider` field is empty string `""` for errors that don't involve a provider (e.g., `unauthorized`)
- `reason` is machine-readable snake_case; `message` is human-readable sentence

**HTTP Status Codes:**
- 400 `bad_request` — malformed/missing required fields
- 401 `unauthorized` — missing or invalid auth
- 422 `unsupported_model` — model not available from any registered adapter
- 503 `provider_unavailable` — adapter health check failed before request forwarded
- 504 `provider_timeout` — adapter did not respond within timeout

**Date/Time:** RFC3339 in all log fields and any structured output

### Adapter Interface Pattern

**The `Provider` interface is the contract all adapters must implement:**
```go
type Provider interface {
    // Chat sends a chat completion request and returns a normalized response.
    Chat(ctx context.Context, req *ChatRequest) (*ChatResponse, error)
    // Health returns nil if the provider is reachable and ready, error otherwise.
    Health(ctx context.Context) error
    // Name returns the registered provider name (e.g., "ollama").
    Name() string
}
```

**Rules:**
- Adapters must never return provider-internal error details — wrap and normalize before returning
- Adapters receive a `context.Context` with the request deadline already set — do not create a new context with a longer timeout
- Adapters must not log credential values or response bodies
- A stub adapter implementing `Provider` must exist and pass contract tests before any new provider adapter begins implementation

### Process Patterns

**Error Handling:**
- Gateway handlers never return raw `error` values to callers — all errors flow through `internal/errors` mapping functions
- Adapter errors are wrapped at the adapter boundary: `fmt.Errorf("ollama: %w", err)` — never surface raw provider messages in HTTP responses
- Panic recovery middleware at the router level — unhandled panics return 500 with a generic message, never a stack trace
- No `log.Fatal` outside of `main()` startup; use structured `slog.Error` + `os.Exit(1)` only during initialization

**Streaming:** `stream: true` is out of scope for MVP. The pipeline handler checks for `stream: true` in the request body after parsing (step 4) and returns `501 Not Implemented` with a `not_supported` error envelope immediately. No provider call is made.

**Request Pipeline Ordering (must not vary):**
1. Panic recovery middleware
2. Request ID injection
3. Auth middleware (Vault token validation — rejects 401 before any further processing)
4. Request body parsing + JSON Schema validation (from `oapi-codegen` generated validators)
4a. Streaming check — if `stream: true`, return 501 immediately
5. Provider resolution (model field → registered adapter)
6. Forward to adapter with context deadline
7. Normalize response / map errors
8. Write response

**Config Loading:**
- All configuration loaded once at startup into a `GatewayConfig` struct
- No config hot-reload for MVP — restart to pick up changes
- Missing required env vars cause startup failure with a descriptive error message listing all missing vars
- No defaults for security-sensitive values (`GATEWAY_VAULT_ADDR`, etc.)

### Enforcement Guidelines

**All AI Agents MUST:**
- Implement the `Provider` interface exactly as defined in `internal/adapter/provider.go` — no additional methods on the public interface
- Route all error responses through `internal/errors` mapping functions — never construct error envelopes inline in handlers
- Use `context.Context` from the inbound request — never create a new background context in handler or adapter code
- Follow the request pipeline ordering above — no re-ordering for "optimization"
- Write table-driven tests for all handler and adapter logic

**Anti-Patterns (never do these):**
- Logging the `Authorization` header value at any log level
- Returning provider error messages directly to callers without normalization
- Adding provider-specific branching in `internal/pipeline` — that belongs in the adapter
- Creating config structs per-package — all config flows from `internal/config.GatewayConfig`

## Project Structure & Boundaries

### Complete Project Directory Structure

```
terminuS-inference-gateway/
├── README.md
├── go.mod
├── go.sum
├── .env.example                        # documented env vars, no secrets
├── .gitignore
├── Makefile                            # build, test, lint, generate targets
├── Dockerfile                          # multi-stage: builder → distroless/static
├── .github/
│   └── workflows/
│       └── ci.yml                      # build, test, contract-test, lint
├── api/
│   ├── openapi.yaml                    # OpenAI-compatible spec — source of truth
│   ├── oapi-codegen.yaml               # codegen config
│   └── gen/                            # generated — do not edit
│       ├── types.gen.go
│       └── server.gen.go
├── cmd/
│   └── gateway/
│       └── main.go                     # wiring only: config → providers → router → server
├── internal/
│   ├── adapter/
│   │   ├── provider.go                 # Provider interface definition
│   │   ├── registry.go                 # provider registration and lookup by model prefix
│   │   ├── stub.go                     # StubProvider — contract test baseline
│   │   ├── stub_test.go
│   │   ├── ollama/
│   │   │   ├── adapter.go              # OllamaAdapter implements Provider
│   │   │   ├── adapter_test.go
│   │   │   └── adapter_integration_test.go
│   │   └── contract/
│   │       └── suite_test.go           # contract test suite run against all registered adapters
│   ├── auth/
│   │   ├── middleware.go               # chi middleware: validates Bearer token via Vault
│   │   ├── vault.go                    # Vault client wrapper
│   │   └── middleware_test.go
│   ├── config/
│   │   ├── config.go                   # GatewayConfig struct
│   │   ├── loader.go                   # env var loading + startup validation
│   │   └── loader_test.go
│   ├── errors/
│   │   ├── errors.go                   # error type constants + ErrorEnvelope struct
│   │   ├── mapper.go                   # adapter error → HTTP status + envelope
│   │   └── errors_test.go
│   ├── health/
│   │   ├── handlers.go                 # /healthz and /readyz handlers
│   │   └── handlers_test.go
│   └── pipeline/
│       ├── handler.go                  # chat completions handler: resolves provider, forwards, normalizes
│       ├── handler_test.go
│       └── normalize.go                # response normalization (finish_reason mapping, field alignment)
└── deploy/
    └── helm/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
            ├── deployment.yaml
            ├── service.yaml
            ├── configmap.yaml
            └── _helpers.tpl
```

### Architectural Boundaries

**API Boundary (inbound):**
- Single ingress: `POST /v1/chat/completions` — all inference requests enter here
- Read-only endpoints: `GET /healthz`, `GET /readyz`
- All inbound requests pass through: panic recovery → request ID → auth middleware → handler
- The `api/gen/` types define the contract; handlers use generated types only

**Adapter Boundary:**
- The `Provider` interface in `internal/adapter/provider.go` is the sole contract between pipeline and adapters
- Adapters live in `internal/adapter/{name}/`; no adapter code outside this tree
- The registry in `internal/adapter/registry.go` owns model-prefix → adapter mapping
- Nothing outside `internal/adapter/` imports adapter-specific packages directly

**Auth Boundary:**
- `internal/auth/` owns all Vault interaction; no other package calls Vault
- Auth middleware runs before any business logic; requests rejected here never reach the pipeline
- The middleware only returns `true/false` (authorized / not); no claims or identity forwarded

**Error Boundary:**
- `internal/errors/mapper.go` is the single path from `error` → HTTP response
- Handlers never construct `ErrorEnvelope` inline; they call `errors.Map(err)` and write the result
- Adapter errors never reach HTTP handlers unwrapped

### Requirements to Structure Mapping

| Requirement | Location |
|-------------|----------|
| FR1–FR3: OpenAI-compatible request/response | `internal/pipeline/handler.go`, `normalize.go` |
| FR4: Model field routing | `internal/adapter/registry.go` |
| FR5–FR6: Provider abstraction, config-driven | `internal/adapter/provider.go`, `registry.go`, `internal/config/` |
| FR7: Adapter readiness pre-check | `internal/adapter/provider.go` (`Health()`), `internal/pipeline/handler.go` |
| FR8: Stubbable adapter interface | `internal/adapter/stub.go` |
| FR9–FR11: Auth enforcement | `internal/auth/middleware.go`, `vault.go` |
| FR12–FR17: Error contract | `internal/errors/errors.go`, `mapper.go` |
| FR18–FR20: Health/readiness endpoints | `internal/health/handlers.go` |
| FR21–FR23: OpenAPI spec + contract tests | `api/openapi.yaml`, `internal/adapter/contract/suite_test.go` |

### Integration Points

**Internal Communication:**
- `pipeline/handler.go` → `adapter/registry.go` → `adapter/{name}/adapter.go` (via `Provider` interface)
- `auth/middleware.go` → `auth/vault.go` → Vault HTTP API
- `health/handlers.go` → `adapter/registry.go` → each registered `Provider.Health()`
- All packages receive `config.GatewayConfig` at construction time from `cmd/gateway/main.go`

**External Integrations:**
- **Vault** (`internal/auth/vault.go`): `GET {GATEWAY_VAULT_ADDR}/v1/auth/token/lookup-self`
- **Ollama** (`internal/adapter/ollama/adapter.go`): `POST {GATEWAY_OLLAMA_BASE_URL}/api/chat`
- Inbound callers: any OpenAI-compatible client (terminus.platform workers, local dev tools)

**Data Flow:**
```
Caller → POST /v1/chat/completions
  → panic recovery middleware
  → request ID middleware
  → auth middleware → Vault lookup-self
  → pipeline handler
      → registry.Lookup(model) → Provider
      → provider.Health(ctx)
      → provider.Chat(ctx, req)
      → normalize.Response(raw) → ChatResponse
  → write response
  ← caller receives OpenAI-compatible response or error envelope
```

### Development Workflow Integration

**Makefile targets:**
- `make generate` — runs `oapi-codegen` from `api/openapi.yaml` into `api/gen/`
- `make build` — `go build -ldflags="-s -w" ./cmd/gateway`
- `make test` — `go test ./...` (unit + integration excluded by default)
- `make test-integration` — `INTEGRATION=true go test ./...`
- `make contract-test` — runs `internal/adapter/contract/suite_test.go` against all registered adapters
- `make lint` — `golangci-lint run`
- `make docker` — multi-stage Docker build
- `make run-local` — runs the gateway locally using the stub provider and a Vault dev-mode server:
  ```bash
  # Requires: vault server -dev running on localhost:8200 (standard dev-mode root token)
  GATEWAY_VAULT_ADDR=http://localhost:8200 \
  GATEWAY_VAULT_TOKEN=root \
  GATEWAY_PROVIDER=stub \
  go run ./cmd/gateway
  ```
  No Ollama required — `GATEWAY_PROVIDER=stub` activates `internal/adapter/stub.go` (StubProvider).

**Build Process:**
- `api/openapi.yaml` is the first artifact — generated types flow from it
- `make generate` must be run after any spec change; generated files committed to repo
- CI runs: lint → unit test → contract test → docker build

## Architecture Validation Results

### Coherence Validation ✅

**Decision Compatibility:**
All technology choices are compatible: Go static binary pairs correctly with `distroless/static` (CA certificates available for Vault TLS). `oapi-codegen` generates types and a `ServerInterface` that `chi` handlers implement directly. `log/slog` is Go standard library — no additional dependency. All versions are current stable releases.

One coherence gap resolved: auth scope clarified to Vault token only for MVP. k8s service account (TokenReview API) is a clean post-MVP addition behind the same `Authenticator` interface — no structural change required when that bridge is crossed.

**Pattern Consistency:**
All patterns are internally consistent. Error flow is unidirectional: adapter → `internal/errors` mapper → HTTP response. No circular dependencies. Config flows one direction: env vars → `GatewayConfig` → all packages via constructor injection. Pipeline ordering is fixed and documented; no package can reorder it without touching `cmd/gateway/main.go`.

**Structure Alignment:**
Project structure directly reflects the six bounded packages. Each package has a single responsibility with no cross-package cycles. `internal/` enforces Go's visibility rules — no adapter-specific package is importable from outside `internal/adapter/`.

### Requirements Coverage Validation ✅

**Functional Requirements (23/23 covered):**
All FR groups have explicit structural support. FR8 (stubbable adapter) is satisfied by `internal/adapter/stub.go` existing before any provider adapter is built — this is the contract validation artifact.

**Auth scope decision recorded:**
- **MVP:** Vault token only (`Authorization: Bearer <vault-token>` → Vault `/v1/auth/token/lookup-self`)
- **Deferred:** k8s service account (TokenReview API) — addable behind `Authenticator` interface without structural change
- **Rationale:** Primary consumers (platform workers) call from inside k8s; Vault token covers dev/operator access from outside. Service account is the cleaner long-term default for pod-to-pod calls but adds no capability needed for MVP.

**Non-Functional Requirements:**
- NFR-P1 (100ms overhead): Go + chi baseline overhead is ~0.2ms; significant headroom
- NFR-P2 (configurable timeout): `GATEWAY_PROVIDER_TIMEOUT_SECONDS` env var, conservative default
- NFR-S1–S4 (security): TLS enforced, credentials never logged, auth on every request, no session cache
- NFR-R1–R3 (reliability): structured errors, clean restart (stateless), per-adapter failure isolation
- NFR-I1–I3 (integration): Vault startup gate, k3s deployment target, env-var provider config

### Implementation Readiness Validation ✅

**Decision Completeness:** All critical decisions documented. Language, framework, spec tooling, auth mechanism, error contract, deployment target, container image, test approach — all specified with rationale.

**Structure Completeness:** Complete directory tree defined to individual file level. All 23 FRs mapped to specific files.

**Pattern Completeness:** 7 conflict points identified and resolved. Adapter interface defined with exact method signatures. Request pipeline ordering fixed. Error flow is single-path. Config injection pattern specified.

### Gap Analysis Results

**Critical gaps:** None remaining.

**Deferred (explicit, not forgotten):**
- k8s service account auth — deferred post-MVP; `Authenticator` interface accommodates it
- Rate limiting — deferred to `terminus-inference-provider-routing` (per PRD scope boundary)
- Service mesh auth delegation — future upgrade path once Istio/Linkerd is in place

### Architecture Completeness Checklist

**✅ Requirements Analysis**
- [x] Project context thoroughly analyzed
- [x] Scale and complexity assessed (high — abstraction correctness, not feature volume)
- [x] Technical constraints identified (OpenAI v1 lock, no provider policy in gateway)
- [x] Cross-cutting concerns mapped (auth, error normalization, health propagation, config)

**✅ Architectural Decisions**
- [x] Language: Go (constitution Article 7 ratified)
- [x] HTTP router: `chi`
- [x] API spec + codegen: `oapi-codegen` from `api/openapi.yaml`
- [x] Auth: Vault token, MVP scope explicit
- [x] Container: `distroless/static`
- [x] Config: env vars, startup validation

**✅ Implementation Patterns**
- [x] Naming conventions: packages, symbols, env vars, log fields
- [x] Adapter interface: exact Go method signatures defined
- [x] Request pipeline ordering: fixed, 8 steps
- [x] Error flow: single path through `internal/errors`
- [x] Test patterns: table-driven, co-located, integration guard

**✅ Project Structure**
- [x] Complete directory tree to file level
- [x] All 6 packages with single responsibilities
- [x] All 23 FRs mapped to specific files
- [x] Data flow diagram
- [x] Makefile targets defined

### Architecture Readiness Assessment

**Overall Status:** READY FOR IMPLEMENTATION

**Confidence Level:** High

**Key Strengths:**
- Adapter interface is the primary deliverable and is fully specified — second provider can be stubbed immediately
- Error contract is complete and single-path — no silent failures possible by construction
- `oapi-codegen` makes the OpenAPI spec the literal source of truth for types and server stubs
- Stateless gateway with env-var config — clean, k8s-native, no operational surprises

**Areas for Future Enhancement:**
- k8s service account auth (post-MVP, no structural change required)
- Request-level telemetry / tracing (add to pipeline as middleware)
- Rate limiting (belongs in `terminus-inference-provider-routing`)

### Implementation Handoff

**AI Agent Guidelines:**
- Follow all architectural decisions exactly as documented
- The `Provider` interface in `internal/adapter/provider.go` is immutable — implement it, do not extend it
- All error responses flow through `internal/errors/mapper.go` — never inline
- Use `context.Context` from the inbound request only — never background context in handlers or adapters
- Refer to this document for all architectural questions before writing code

**First Implementation Priority:**
```bash
# 1. Initialize module
go mod init github.com/electricm0nk/terminus-inference-gateway
# 2. Write api/openapi.yaml (OpenAI-compatible spec)
# 3. Run: make generate
# 4. Define internal/adapter/provider.go (Provider interface)
# 5. Implement internal/adapter/stub.go + contract test suite
```
