---
stepsCompleted: [step-01-validate-prerequisites, step-02-design-epics, step-03-create-stories, step-04-final-validation]
inputDocuments:
  - docs/terminus/inference/gateway/prd.md
  - docs/terminus/inference/gateway/architecture.md
---

# terminus-inference-gateway - Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for `terminus-inference-gateway`, decomposing requirements from the PRD and Architecture into implementable stories.

## Requirements Inventory

### Functional Requirements

FR1: A calling service can submit an OpenAI-compatible chat completions request and receive an OpenAI-compatible response
FR2: The gateway normalizes provider response fields into the OpenAI chat completion schema before returning to the caller
FR3: The gateway propagates `finish_reason` (normalized to `stop`, `length`, or `content_filter`) in every successful response
FR4: Callers target a specific model using the standard OpenAI `model` field (e.g., `ollama/mistral`)
FR5: The gateway routes inference requests to a registered provider adapter without exposing provider-specific details to callers
FR6: An operator can register a new provider adapter by implementing the adapter interface and adding it to config — no handler code changes required
FR7: The gateway validates provider adapter readiness before forwarding any request
FR8: The adapter interface can be exercised with a stub implementation independent of a live provider
FR9: All requests must present a Vault token or k8s service account credential to be processed
FR10: The gateway validates caller credentials before resolving a provider or forwarding any request
FR11: Requests with missing or invalid credentials are rejected at the gateway boundary with no provider call made
FR12: All error responses use a structured envelope with `type`, `provider`, `reason`, and `message` fields
FR13: Malformed or incomplete requests are rejected with a `400` before provider resolution
FR14: Requests for models unavailable from any registered adapter return `422` with type `unsupported_model`
FR15: Provider health check failures return `503` with type `provider_unavailable` identifying the failing provider
FR16: Provider response timeouts return `504` with type `provider_timeout`
FR17: Auth failures return `401` with type `unauthorized`
FR18: The gateway exposes a liveness endpoint indicating process health
FR19: The gateway exposes a readiness endpoint indicating provider adapter health
FR20: The readiness endpoint reflects current health state of all registered adapters
FR21: The gateway API contract is expressed as an OpenAPI specification collocated with the implementation
FR22: The OpenAPI spec is the source-of-truth for provider adapter contract tests
FR23: Contract tests validate request schema, response schema, error shape, and normalization guarantees across all registered adapters

### NonFunctional Requirements

NFR-P1: Gateway overhead (excluding provider processing time) must not exceed 100ms per request under normal operating conditions
NFR-P2: Gateway timeout threshold for provider responses is configurable; default conservatively 5 minutes for unattended batch workloads
NFR-P3: Health and readiness endpoint responses must complete within 1 second
NFR-S1: All inter-service communication uses TLS — no plaintext credential or response data in transit
NFR-S2: Auth credentials are never logged, included in error responses, or forwarded to provider adapters
NFR-S3: Provider-internal error details (stack traces, internal hostnames, account info) are not exposed in gateway error responses — only normalized error envelopes returned to callers
NFR-S4: Vault token or k8s service account validation performed on every request — no session caching or token reuse between requests
NFR-R1: Provider failures are detected and returned as structured errors within the gateway timeout threshold — no silent hangs or raw socket errors exposed to callers
NFR-R2: Gateway process must restart cleanly after a crash without requiring manual state cleanup
NFR-R3: A provider adapter failure must not affect gateway availability for requests targeting other registered adapters
NFR-I1: The gateway depends on `terminus-infra-secrets` for Vault/k8s credential validation — declared and tested as a startup gate
NFR-I2: The gateway depends on `terminus-infra-k3s` for deployment substrate — infrastructure readiness is a deployment prerequisite, not a runtime concern
NFR-I3: The Ollama adapter must be configurable via environment variable or config file — no hardcoded provider URLs or credentials in source

### Additional Requirements

- **Starter scaffold (Epic 1, Story 1):** Go standard library + `chi` router. Initialize with: `go mod init github.com/electricm0nk/terminus-inference-gateway`, add `chi`, add `oapi-codegen`. This is the required first story.
- **oapi-codegen config:** `oapi-codegen.yaml` committed alongside OpenAPI spec; generates Go types and `StrictServerInterface` stubs into `api/gen/` — generated files must never be manually edited
- **Package structure enforced:** `/cmd/gateway`, `/internal/adapter`, `/internal/auth`, `/internal/config`, `/internal/errors`, `/internal/health`, `/internal/pipeline`, `/api/`, `/deploy/helm/`
- **Provider interface canonical definition:** `Chat(ctx, *ChatRequest) (*ChatResponse, error)`, `Health(ctx) error`, `Name() string` — must not deviate
- **Request pipeline ordering locked:** Panic recovery → Request ID → Auth middleware → Body parsing/schema validation → Streaming check (501) → Provider resolution → Forward to adapter → Normalize → Write response
- **Stub provider required first:** `StubProvider` in `internal/adapter/stub.go` must exist and pass contract tests before any live adapter implementation begins
- **Streaming → 501:** `stream: true` in request body returns `501 Not Implemented` after body parsing, before provider resolution
- **Config behavior:** All config loaded once at startup into `GatewayConfig`; missing required env vars cause descriptive startup failure listing all missing vars; no hot-reload
- **Vault startup gate:** Gateway refuses to start if Vault is unreachable at startup
- **Container:** `gcr.io/distroless/static` base image (CA certs available, no shell)
- **Helm chart:** Co-located in gateway repo at `/deploy/helm/` — not in terminus-infra
- **CI pipeline:** `.github/workflows/ci.yml` — build, test, contract-test, lint
- **Test patterns:** Table-driven tests for all handler and adapter logic; integration tests: `_integration_test.go` suffix, skipped unless `INTEGRATION=true`
- **Observability TODO (deferred post-MVP):** Domain constitution Art.10 requires route_id, duration, token counts, outcome, cost_tag per request — must be addressed before production readiness; out of scope for this feature
- **Error normalization:** All errors flow through `internal/errors` mapping functions — never constructed inline in handlers; adapters wrap provider errors before returning

### FR Coverage Map

FR1: Epic 2 - OpenAI-compatible request accepted and response returned
FR2: Epic 2 - Provider response normalized to OpenAI chat completion schema
FR3: Epic 2 - finish_reason propagated (stop / length / content_filter)
FR4: Epic 2 - Caller targets model via standard OpenAI `model` field
FR5: Epic 2 - Gateway routes to registered adapter without exposing provider details
FR6: Epic 3 - New adapter registered via config only, no handler code changes
FR7: Epic 2 - Provider adapter readiness validated before request is forwarded
FR8: Epic 3 - Adapter interface exercisable with StubProvider independent of live provider
FR9: Epic 1 - All requests require Vault token or k8s service account credential
FR10: Epic 1 - Credentials validated before provider resolution or request forwarding
FR11: Epic 1 - Missing/invalid credentials rejected at gateway boundary, no provider call made
FR12: Epic 1 - All error responses use structured envelope (type, provider, reason, message)
FR13: Epic 1 - Malformed/incomplete requests rejected 400 before provider resolution
FR14: Epic 2 - Unavailable model returns 422 unsupported_model
FR15: Epic 2 - Provider health check failure returns 503 provider_unavailable
FR16: Epic 2 - Provider timeout returns 504 provider_timeout
FR17: Epic 1 - Auth failure returns 401 unauthorized
FR18: Epic 1 - Liveness endpoint /healthz exposes process health
FR19: Epic 1 - Readiness endpoint /readyz exposes provider adapter health
FR20: Epic 1 - /readyz reflects current health state of all registered adapters
FR21: Epic 1 - OpenAPI spec collocated with implementation (scaffold established)
FR22: Epic 2 - OpenAPI spec is source-of-truth for contract tests (single adapter)
FR23: Epic 3 - Contract tests run against all registered adapters (multi-adapter coverage)

## Epic List

### Epic 1: Authenticated, Observable Gateway Foundation
An operator can stand up a running, auth-enforcing gateway that rejects unauthenticated requests at the boundary, returns structured error envelopes for all auth and malformed-request failure conditions, exposes accurate health and readiness endpoints, and has its API contract expressed as a colocated OpenAPI specification. No inference logic is present yet — the request surface is wired, enforcing, and observable.
**FRs covered:** FR9, FR10, FR11, FR12, FR13, FR17, FR18, FR19, FR20, FR21
**NFRs addressed:** NFR-S1, NFR-S2, NFR-S4 (auth/TLS), NFR-P3 (health latency), NFR-R2 (clean restart), NFR-I1 (Vault startup gate), NFR-I2 (k3s deployment), NFR-I3 (env config)

### Epic 2: Ollama Inference Request Fulfillment
A platform service author can submit OpenAI-compatible chat completion requests and receive normalized inference responses. Provider health check failures, timeouts, and unsupported models all return structured errors. The adapter contract is validated by a contract test suite driven from the OpenAPI spec against the Ollama adapter.
**FRs covered:** FR1, FR2, FR3, FR4, FR5, FR7, FR14, FR15, FR16, FR22, FR23 (single-adapter baseline)
**NFRs addressed:** NFR-P1 (100ms overhead), NFR-P2 (configurable timeout), NFR-R1 (no silent hangs), NFR-R3 (adapter isolation), NFR-S3 (no provider internals leaked)

### Epic 3: Config-Driven Provider Extensibility
An operator can register a second provider adapter by implementing the `Provider` interface and adding config only — no gateway handler code changes required. A `StubProvider` implementation exists and passes contract tests, validating the adapter interface independently of any live provider. The contract test suite runs against multiple adapters, confirming the abstraction is genuine.
**FRs covered:** FR6, FR8, FR23 (multi-adapter extension)
**Architecture deliverables:** StubProvider, adapter registry model-prefix lookup, contract suite extended to stub P2, CI pipeline executing contract tests

---

## Epic 1: Authenticated, Observable Gateway Foundation

An operator can stand up a running, auth-enforcing gateway that rejects unauthenticated requests at the boundary, returns structured error envelopes for all auth and malformed-request failure conditions, exposes accurate health and readiness endpoints, and has its API contract expressed as a colocated OpenAPI specification. No inference logic is present yet — the request surface is wired, enforcing, and observable.

### Story 1.1: Initialize Go Module Scaffold

As a **developer**,
I want a properly initialized Go module with the correct project structure, build tooling, and scaffolding files,
So that all subsequent implementation stories have a consistent, architecture-conformant foundation to build on.

**Acceptance Criteria:**

**Given** the gateway repository is empty
**When** the scaffold story is executed
**Then** `go.mod` exists with module path `github.com/electricm0nk/terminus-inference-gateway`
**And** `github.com/go-chi/chi/v5` is added as a dependency in `go.mod`
**And** `github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen` is added as a tool dependency
**And** all directories match the architecture-specified layout: `/cmd/gateway`, `/internal/adapter`, `/internal/auth`, `/internal/config`, `/internal/errors`, `/internal/health`, `/internal/pipeline`, `/api`, `/deploy/helm`
**And** `Makefile` exists with targets: `build`, `test`, `lint`, `generate`
**And** `.env.example` documents all required env vars with placeholder values and no secrets
**And** `.gitignore` excludes compiled binaries, `/api/gen/`, and env files containing secrets
**And** `README.md` contains module name, quick-start instructions, and a link to the OpenAPI spec

---

### Story 1.2: OpenAPI Spec and Code Generation

As a **platform service author**,
I want the gateway's API contract expressed as an OpenAPI 3.0 specification with generated Go types and server stubs,
So that the route shapes, request/response schemas, and error envelopes are machine-verifiable and the server interface is compile-time enforced from day one.

**Acceptance Criteria:**

**Given** the scaffold from Story 1.1 exists
**When** the OpenAPI spec is authored and codegen is configured
**Then** `api/openapi.yaml` defines `POST /v1/chat/completions`, `GET /healthz`, and `GET /readyz`
**And** `api/openapi.yaml` includes the full error envelope schema (`type`, `provider`, `reason`, `message` fields)
**And** `api/openapi.yaml` includes OpenAI-compatible chat completion request and response schemas
**And** `api/oapi-codegen.yaml` is committed with `strict-server: true` and output targeting `api/gen/`
**And** `make generate` runs `oapi-codegen` and produces `api/gen/types.gen.go` and `api/gen/server.gen.go` without error
**And** generated files include a `// Code generated` header and ARE committed to the repository — `api/gen/` is NOT in `.gitignore` (CI depends on detecting stale generated files via git diff)
**And** a chi router in `cmd/gateway/main.go` wires the generated `StrictServerInterface` with stub handlers
**And** stub handlers for inference routes return `501 Not Implemented` with a `not_supported` error envelope
**And** `go build ./...` succeeds with no errors

---

### Story 1.3: Configuration Loader

As an **operator**,
I want the gateway to validate all required environment variables at startup and fail with a descriptive message listing every missing variable,
So that misconfigured deployments fail immediately and clearly rather than silently misbehaving at runtime.

**Acceptance Criteria:**

**Given** the scaffold and OpenAPI gen from Stories 1.1–1.2 exist
**When** the configuration loader is implemented
**Then** `internal/config/config.go` defines `GatewayConfig` as a single struct containing all gateway configuration fields
**And** `internal/config/loader.go` reads all config from environment variables at startup
**And** when any required env var is absent, startup fails with a single descriptive error message listing all missing variables by name
**And** security-sensitive vars (`GATEWAY_VAULT_ADDR`, `GATEWAY_VAULT_TOKEN`) have no default values and cause startup failure if absent
**And** non-sensitive vars (`GATEWAY_PORT`, `GATEWAY_LOG_LEVEL`) have documented defaults
**And** `GATEWAY_OLLAMA_BASE_URL` and `GATEWAY_OLLAMA_TIMEOUT_SECONDS` are defined (used in Epic 2)
**And** `internal/config/loader_test.go` includes table-driven tests covering: all vars present, missing required var, multiple missing required vars
**And** `GatewayConfig` is populated once at startup and passed by value — no global config state

---

### Story 1.4: Error Envelope Package

As a **gateway pipeline**,
I want a centralized error package with typed constants and mapper functions,
So that all error responses flow through a single normalization path and no handler or adapter constructs error envelopes inline.

**Acceptance Criteria:**

**Given** the scaffold from Story 1.1 exists
**When** the error envelope package is implemented
**Then** `internal/errors/errors.go` defines the `ErrorEnvelope` struct with `Type`, `Provider`, `Reason`, and `Message` string fields
**And** error type string constants are defined for: `bad_request`, `unauthorized`, `unsupported_model`, `provider_unavailable`, `provider_timeout`, `not_supported`
**And** `internal/errors/mapper.go` provides mapper functions that accept an error type constant and return the correct HTTP status code
**And** a constructor function `NewEnvelope(type, provider, reason, message string) ErrorEnvelope` is defined
**And** `provider` field is empty string `""` for errors that do not involve a provider (e.g., `unauthorized`)
**And** `internal/errors/errors_test.go` contains table-driven tests covering each error type → HTTP status mapping
**And** no handler or adapter file constructs an `ErrorEnvelope` struct literal directly — all construction goes through the constructor

---

### Story 1.5: Vault Auth Middleware

As a **platform service author**,
I want the gateway to reject requests without valid Vault tokens before any provider interaction or resource consumption,
So that unauthorized callers cannot trigger inference workloads and credentials are never exposed through gateway responses or logs.

**Acceptance Criteria:**

**Given** the config loader (1.3) and error envelope package (1.4) exist
**When** the Vault auth middleware is implemented and wired into the chi router
**Then** `internal/auth/vault.go` implements a Vault client that calls `/v1/auth/token/lookup-self` to validate a token
**And** `internal/auth/middleware.go` implements a chi middleware that extracts the `Authorization: Bearer <token>` header
**And** requests missing the `Authorization` header are rejected with `401` and `unauthorized` error envelope before reaching any handler
**And** requests with a token that fails Vault lookup are rejected with `401` and `unauthorized` error envelope
**And** the Authorization header value is never written to logs at any log level
**And** Vault token validation is performed on every request with no session caching (NFR-S4)
**And** at gateway startup, the Vault client attempts to reach `GATEWAY_VAULT_ADDR`; if unreachable, startup fails with a descriptive error (NFR-I1)
**And** `internal/auth/middleware_test.go` includes table-driven tests covering: valid token, missing header, invalid token, Vault unreachable
**And** the middleware is positioned as the first handler after request ID injection in the chi router — before body parsing or provider resolution

---

### Story 1.6: Health and Readiness Handlers

As an **operator**,
I want `/healthz` and `/readyz` endpoints that respond within 1 second,
So that the k3s cluster can accurately determine gateway liveness and readiness for traffic routing.

**Acceptance Criteria:**

**Given** the chi router and error envelope package from Stories 1.2 and 1.4 exist
**When** the health handlers are implemented and registered on the router
**Then** `GET /healthz` returns `200 OK` with a JSON body `{"status": "ok"}` when the gateway process is running
**And** `GET /readyz` returns `200 OK` with a JSON body `{"status": "ok", "adapters": {}}` when no adapters are registered (baseline — adapters added in Epic 2)
**And** both endpoints respond within 1 second under normal operating conditions (NFR-P3)
**And** health endpoints do not require an `Authorization` header — they are exempt from auth middleware
**And** `internal/health/handlers_test.go` contains table-driven tests covering: liveness returns 200, readiness returns 200 with empty adapter map
**And** health handler registration in the chi router is outside the authenticated route group

---

### Story 1.7: Container and Deployment Packaging

As an **operator**,
I want a container image and Helm chart for the gateway,
So that it can be deployed to the k3s cluster and serve health endpoints with auth enforcement before any inference logic is present.

**Acceptance Criteria:**

**Given** all Stories 1.1–1.6 are complete and the gateway builds and serves health/auth endpoints
**When** the container and deployment packaging is implemented
**Then** `Dockerfile` uses a multi-stage build: Go builder stage produces a static binary; final stage is `gcr.io/distroless/static`
**And** `go build -ldflags="-s -w"` is used in the builder stage to produce a minimal binary
**And** `make docker-build` builds the image without error
**And** the Helm chart at `/deploy/helm/` includes `Chart.yaml`, `values.yaml`, and templates for `Deployment`, `Service`, and `ConfigMap`
**And** `values.yaml` exposes: `vault.addr`, `vault.tokenSecret` (k8s Secret name), `providers.ollama.enabled`, `providers.ollama.baseURL`, `providers.ollama.timeoutSeconds`, `gateway.port`, `gateway.logLevel`
**And** the Helm chart mounts the Vault token from the named k8s Secret as `GATEWAY_VAULT_TOKEN` env var via `secretKeyRef`
**And** a `make helm-lint` target runs `helm lint deploy/helm/` without errors
**And** when deployed with valid Vault config, the gateway starts, serves `GET /healthz` → 200, and rejects unauthenticated `POST /v1/chat/completions` with 401

---

## Epic 2: Ollama Inference Request Fulfillment

A platform service author can submit OpenAI-compatible chat completion requests and receive normalized inference responses. Provider health check failures, timeouts, and unsupported models all return structured errors. The adapter contract is validated by a contract test suite driven from the OpenAPI spec against the Ollama adapter.

### Story 2.1: Provider Interface Definition

As a **developer**,
I want a canonical `Provider` interface and associated request/response types defined in a single location,
So that all adapter implementations are compile-time enforced against the same contract and no handler code couples to provider-specific types.

**Acceptance Criteria:**

**Given** the scaffold from Epic 1 exists
**When** the provider interface is implemented
**Then** `internal/adapter/provider.go` defines the `Provider` interface with exactly three methods: `Chat(ctx context.Context, req *ChatRequest) (*ChatResponse, error)`, `Health(ctx context.Context) error`, and `Name() string`
**And** `ChatRequest` struct contains at minimum: `Model string`, `Messages []Message`, `Temperature *float64`, `MaxTokens *int`, `Stream bool`
**And** `ChatResponse` struct contains at minimum: `ID string`, `Model string`, `Choices []Choice`, `Usage Usage`
**And** `Choice` struct contains `Message Message` and `FinishReason string`
**And** `FinishReason` is normalized to one of three values: `"stop"`, `"length"`, `"content_filter"` — no provider-specific values permitted
**And** no additional methods are added to the `Provider` interface beyond the three specified
**And** the interface definition is the only file in `internal/adapter/provider.go` — no adapter implementations in this file

---

### Story 2.2: Adapter Registry

As a **gateway pipeline**,
I want a registry that resolves a model name to its registered provider adapter,
So that routing to the correct adapter requires only a config-driven lookup with no provider-specific branching in handler code.

**Acceptance Criteria:**

**Given** the `Provider` interface from Story 2.1 exists
**When** the adapter registry is implemented
**Then** `internal/adapter/registry.go` defines a `Registry` struct that holds a map of provider name → `Provider`
**And** `Registry.Register(adapter Provider)` registers an adapter using `adapter.Name()` as the key
**And** `Registry.Resolve(model string) (Provider, error)` extracts the provider prefix from the model string (e.g., `"ollama/mistral"` → `"ollama"`) and returns the matching adapter
**And** when no adapter matches the model prefix, `Resolve` returns an error that the pipeline maps to `422 unsupported_model` (FR14)
**And** `registry_test.go` includes table-driven tests covering: known model resolves to correct adapter, unknown model prefix returns error, empty model string returns error
**And** no provider name strings are hardcoded outside of adapter `Name()` implementations — registry is fully data-driven

---

### Story 2.3: Ollama Adapter

As a **platform service author**,
I want the gateway to forward chat completion requests to Ollama and return normalized OpenAI-compatible responses,
So that I can call a single consistent endpoint without knowledge of Ollama's API or response format.

**Acceptance Criteria:**

**Given** the `Provider` interface (2.1) and config loader (1.3) exist
**When** the Ollama adapter is implemented
**Then** `internal/adapter/ollama/adapter.go` defines `OllamaAdapter` implementing the `Provider` interface
**And** `OllamaAdapter.Name()` returns `"ollama"`
**And** `OllamaAdapter.Chat` translates the `ChatRequest` into an Ollama API request, calls `GATEWAY_OLLAMA_BASE_URL/api/chat`, and translates the response back into a `ChatResponse` with normalized `FinishReason`
**And** the request context deadline is respected — `OllamaAdapter.Chat` does not create a new context with a longer timeout
**And** when Ollama does not respond within the context deadline, the adapter returns an error that the pipeline maps to `504 provider_timeout` (NFR-P2, FR16)
**And** `OllamaAdapter.Health` calls `GATEWAY_OLLAMA_BASE_URL/api/tags` and returns nil on 200, error otherwise; the choice of `/api/tags` over other Ollama endpoints is documented in a comment in `adapter.go` with rationale (e.g., availability in all Ollama versions tested)
**And** Ollama-internal error messages are wrapped and normalized before being returned — no raw Ollama response bodies reach the gateway error envelope (NFR-S3)
**And** `internal/adapter/ollama/adapter_test.go` uses table-driven tests with an `httptest.Server` covering: successful response, Ollama 500, network timeout, unexpected response shape
**And** `internal/adapter/ollama/adapter_integration_test.go` exists with `//go:build integration` tag, skipped unless `INTEGRATION=true` env var is set

---

### Story 2.4: Request Pipeline

As a **platform service author**,
I want a complete, ordered request pipeline that enforces auth, validates the request, resolves a provider, and returns a normalized response or structured error,
So that the gateway correctly handles the full inference request lifecycle end-to-end.

**Acceptance Criteria:**

**Given** the auth middleware (1.5), error envelope package (1.4), adapter registry (2.2), and Ollama adapter (2.3) exist
**When** the request pipeline is implemented and wired to the chi router
**Then** `internal/pipeline/handler.go` implements the chat completions handler following the locked pipeline order: panic recovery → request ID injection → auth middleware → body parsing + schema validation → streaming check → provider resolution → forward to adapter → normalize → write response
**And** when `stream: true` is present in the request body, the handler returns `501 Not Implemented` with `not_supported` error envelope immediately after parsing — no adapter call is made
**And** when the request body fails JSON schema validation, the handler returns `400 bad_request` before provider resolution (FR13)
**And** adapter errors are mapped through `internal/errors` mapper functions — no inline error envelope construction in the pipeline handler
**And** the pipeline handler records request timing via structured `slog` fields (`duration_ms` split into pre-adapter and post-adapter) on every request — this provides the observability baseline for NFR-P1 (100ms overhead budget); no unit-test assertion on wall-clock time is required
**And** `internal/pipeline/handler_test.go` uses table-driven tests with a mock `Provider` covering: successful round-trip, missing auth (401), invalid body (400), unknown model (422), provider unavailable — mock `Chat()` returns `ErrProviderUnavailable` (503), provider timeout — mock `Chat()` returns `ErrProviderTimeout` (504), streaming request (501)
**And** the 503 and 504 test cases use a mock `Provider` whose `Chat()` method returns the corresponding error directly — they do not depend on any prior health state
**And** the panic recovery middleware returns `500` with a generic message on unhandled panics — no stack traces in responses
**And** `POST /v1/chat/completions` with a valid Vault token and `{"model": "ollama/mistral", "messages": [...]}` returns an OpenAI-compatible chat completion response

---

### Story 2.5: Contract Test Suite (Single Adapter)

As a **developer**,
I want a contract test suite that validates request schema, response schema, error shapes, and `finish_reason` normalization against the Ollama adapter,
So that the adapter's conformance to the `Provider` interface and the OpenAPI spec is automatically verified and regressions are caught at test time.

**Acceptance Criteria:**

**Given** the Ollama adapter (2.3), generated OpenAPI types (1.2), and `Provider` interface (2.1) exist
**When** the contract test suite is implemented
**Then** `internal/adapter/contract/suite_test.go` defines a reusable test suite that accepts any `Provider` implementation
**And** the suite validates: a well-formed `ChatRequest` produces a `ChatResponse` matching the OpenAI response schema, `FinishReason` is one of `"stop"`, `"length"`, `"content_filter"`, error responses use the normalized error envelope shape, `Health()` returns nil when the provider is reachable
**And** the suite is run against `OllamaAdapter` in `internal/adapter/ollama/adapter_integration_test.go` (requires `INTEGRATION=true`)
**And** `make test` runs all non-integration tests, including contract suite with a mock `Provider`
**And** `make contract-test` runs the integration-tagged contract suite (requires live Ollama)
**And** the contract test file references generated OpenAPI types from `api/gen/` for schema assertions — not hand-coded structs

---

### Story 2.6: Readiness Endpoint — Live Adapter Health

As an **operator**,
I want `/readyz` to reflect the live health state of all registered provider adapters,
So that the cluster can gate traffic accurately and adapter failures are surfaced immediately without affecting gateway liveness.

**Acceptance Criteria:**

**Given** the health handlers (1.6), Ollama adapter (2.3), and adapter registry (2.2) exist
**When** the readiness handler is updated to query registered adapters
**Then** `GET /readyz` calls `Health(ctx)` on each registered adapter and includes per-adapter status in the response
**And** when all adapters are healthy, `/readyz` returns `200 OK` with `{"status": "ok", "adapters": {"ollama": "ok"}}`
**And** when one adapter is unhealthy, `/readyz` returns `503` with `{"status": "degraded", "adapters": {"ollama": "unhealthy: <reason>"}}`
**And** `/healthz` always returns `200 OK` regardless of adapter health state — liveness and readiness are independent (NFR-R3)
**And** `/readyz` health check calls complete within 1 second (NFR-P3); a per-adapter timeout is applied to prevent slow health checks from blocking the endpoint
**And** `internal/health/handlers_test.go` is extended with table-driven tests covering: all adapters healthy → 200, one adapter unhealthy → 503, health check timeout → 503

---

## Epic 3: Config-Driven Provider Extensibility

An operator can register a second provider adapter by implementing the `Provider` interface and adding config only — no gateway handler code changes required. A `StubProvider` implementation exists and passes contract tests, validating the adapter interface independently of any live provider. The contract test suite runs against multiple adapters, confirming the abstraction is genuine.

### Story 3.1: StubProvider Implementation

As a **developer**,
I want a `StubProvider` that implements the `Provider` interface with configurable responses and health state,
So that the adapter contract can be validated without a live provider and any future adapter has a proven reference for what correct conformance looks like.

**Acceptance Criteria:**

**Given** the `Provider` interface (2.1) and adapter registry (2.2) exist
**When** the `StubProvider` is implemented
**Then** `internal/adapter/stub.go` defines `StubProvider` implementing the full `Provider` interface (`Chat`, `Health`, `Name`)
**And** `StubProvider.Name()` returns `"stub"`
**And** `StubProvider` accepts configurable `ChatResponse` values and health state at construction time — no hardcoded responses
**And** `StubProvider` can be registered in the adapter registry via `Registry.Register(stub)` with no code changes to the registry or pipeline
**And** when `StubProvider` is the only registered adapter, `POST /v1/chat/completions` with `model: "stub/test"` returns a valid OpenAI-compatible response (end-to-end smoke test)
**And** `internal/adapter/stub_test.go` includes table-driven tests covering: `Chat` returns configured response, `Chat` returns configured error, `Health` returns nil when configured healthy, `Health` returns error when configured unhealthy
**And** `StubProvider` produces responses that pass the full contract test suite without modification to the suite

---

### Story 3.2: Contract Suite Extended to Multiple Adapters

As a **developer**,
I want the contract test suite to run against both `OllamaAdapter` and `StubProvider`,
So that the adapter interface abstraction is proven genuine — a second adapter can be validated with the same suite, with no suite changes.

**Acceptance Criteria:**

**Given** the contract test suite (2.5) and `StubProvider` (3.1) exist
**When** the contract suite is extended for multi-adapter coverage
**Then** `internal/adapter/contract/suite_test.go` runs the full contract suite against `StubProvider` with no modifications to the suite itself
**And** the suite is parameterized — adding a new adapter to the test run requires only adding an entry to the adapter list, not modifying any test logic
**And** `make test` executes the contract suite against `StubProvider` without requiring `INTEGRATION=true` (stub requires no live provider)
**And** `make contract-test` executes the contract suite against both `StubProvider` and `OllamaAdapter` (Ollama requires live provider via `INTEGRATION=true`)
**And** the suite output clearly identifies which adapter is under test for each assertion
**And** all 23 FRs are traceable to at least one passing test across the combined test suite (unit tests, handler tests, and adapter contract suite) — FR9–FR17 are covered by handler tests from Story 2.4; FR1–FR8, FR22–FR23 are covered by adapter contract tests

---

### Story 3.3: CI Pipeline

As a **developer**,
I want a CI pipeline that runs build, unit tests, lint, and contract tests on every push,
So that regressions in the adapter interface, error contract, or OpenAPI conformance are caught automatically before merge.

**Acceptance Criteria:**

**Given** all Epic 1–3 stories are complete
**When** the CI pipeline is implemented
**Then** `.github/workflows/ci.yml` defines a workflow triggered on push to all branches and on pull requests
**And** the pipeline runs the following jobs in order: `generate` (`make generate` — verifies codegen is not stale), `build` (`go build ./...`), `lint` (`golangci-lint run`), `test` (`make test` — unit + contract suite against StubProvider)
**And** the `generate` job fails if `make generate` produces any diff in `api/gen/` — generated files must be committed and current
**And** `make test` in CI does not require `INTEGRATION=true` — all CI-required tests run against stubs only
**And** a `golangci-lint` config file (`.golangci.yml`) is committed enabling at minimum: `govet`, `errcheck`, `staticcheck`, `godot`
**And** the `lint` job uses `golangci-lint-action` with caching enabled (`~/.cache/golangci-lint`) to ensure consistent performance on GitHub Actions runners
**And** the pipeline status badge is referenced in `README.md`
