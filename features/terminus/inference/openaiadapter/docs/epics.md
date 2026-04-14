---
stepsCompleted: [step-01-validate-prerequisites, step-02-design-epics, step-03-create-stories]
inputDocuments:
  - docs/terminus/inference/openaiadapter/prd.md
  - docs/terminus/inference/openaiadapter/architecture.md
  - docs/terminus/inference/openaiadapter/tech-decisions.md
project: terminus-inference-openaiadapter
phase: devproposal
date: '2026-04-07'
---

# terminus-inference-openaiadapter - Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for `terminus-inference-openaiadapter`, decomposing the requirements from the PRD and Architecture into implementable stories.

## Requirements Inventory

### Functional Requirements

FR1: The gateway can route inference requests to an OpenAI-backed provider via a registered OpenAIAdapter
FR2: The OpenAIAdapter maps the gateway's internal ChatRequest to an OpenAI /v1/chat/completions HTTP request
FR3: The OpenAIAdapter maps the OpenAI response to the gateway's internal ChatResponse, including InputTokens and OutputTokens
FR4: The OpenAIAdapter maps a 401 response from OpenAI to a structured error envelope with type: unauthorized
FR5: The OpenAIAdapter maps a 429 response from OpenAI to a structured error envelope with type: rate_limited
FR6: The OpenAIAdapter maps 5xx responses to a structured error envelope with type: provider_error
FR7: The OpenAIAdapter maps request timeouts to a structured error envelope with type: timeout
FR8: The operator can register the OpenAI provider in routing-config.yaml with a priority position, model target, and budget ceiling
FR9: The routing engine selects the OpenAI provider as a fallback when higher-priority providers are unavailable or exhausted
FR10: The routing engine restricts OpenAI provider selection to requests with workloadClass: interactive only
FR11: The gateway response includes degraded: true and fallback_provider: openai when the OpenAI adapter services a fallback request
FR12: The operator can store the OpenAI API key in Vault under the gateway's secret path
FR13: An ESO ExternalSecret syncs the Vault secret into the gateway pod's environment as GATEWAY_OPENAI_API_KEY
FR14: The OpenAIAdapter reads the API key from the pod environment variable at runtime without it appearing in source code, config files, or logs
FR15: The gateway Helm chart passes GATEWAY_OPENAI_API_KEY into the pod environment via secretKeyRef in env: block
FR16: The gateway logs a startup confirmation that the OpenAI adapter is registered at boot time
FR17: The OpenAIAdapter emits a structured log entry per request containing: route_id, provider, model, duration_ms, input_tokens, output_tokens, outcome, and cost_tag
FR18: The structured log entry outcome field reflects the result: success, error, rate_limited, or unauthorized
FR19: The contract test suite includes coverage for the OpenAIAdapter and passes as a gate (inference Art. 2)
FR20: An operator can run a smoke test that produces a non-error response from the OpenAI adapter against the deployed gateway using the ESO-delivered key
FR21: A resilience test validates that when Ollama is unavailable, the routing engine selects OpenAI and the gateway response contains degraded: true
FR22: An Art. 15 AI safety exception artifact containing all 7 required items is committed to the repository before the initiative is promoted to medium audience — COMPLETE
FR23: Batch workloads are prohibited from routing to the OpenAI provider — the routing policy enforces allowedClasses: [interactive] at the config layer

### NonFunctional Requirements

NFR-P1: The adapter must not contribute more than 50ms of processing overhead beyond OpenAI API response time (network latency excluded)
NFR-P2: Request context, including the API key reference, must not be retained in memory after the response is returned
NFR-S1: GATEWAY_OPENAI_API_KEY must never appear in log output, error messages, stack traces, or HTTP responses at any log level
NFR-S2: The API key must only be sourced from the pod environment variable — not from config files, Helm values files, or hardcoded in source
NFR-S3: All communication with the OpenAI API must use TLS — no plaintext HTTP permitted
NFR-S4: The Art. 15 exception artifact must be reviewed and renewed annually; the expiry date must be explicit in the artifact
NFR-R1: On transient OpenAI errors (5xx, timeout), the adapter propagates the structured error to the routing engine — retry and fallback decisions belong to the routing layer, not the adapter
NFR-R2: The adapter must not cache responses — every request is a fresh call
NFR-O1: Structured telemetry logs (FR17) must be emitted at INFO level
NFR-D1: Only workloadClass: interactive requests may be forwarded to OpenAI — batch workload data must not leave local infrastructure (Terminus Art. 7)
NFR-D2: User prompt content forwarded to OpenAI must not be logged by the adapter — telemetry captures metadata only

### Additional Requirements

- **Brownfield extension — no starter template.** The adapter is added to the existing terminus-inference-gateway repository. No new repo, no scaffold.
- **Interface ground truth:** Provider interface with Chat(ctx context.Context, req *ChatRequest) (*ChatResponse, error), Health(ctx context.Context) error, Name() string — from internal/adapter/provider.go
- **Constructor pattern:** func New(apiKey string) (*OpenAIAdapter, error) — validates non-empty key at construction. Returns error if empty.
- **Registry key:** "openai" (via Name()) — model strings take the form openai/gpt-4o-mini
- **Key delivery confirmed:** secretKeyRef in env: block (not extraEnv) — from deploy/helm/templates/deployment.yaml
- **Error enum (binding):** unauthorized / rate_limited / provider_error / timeout with reason values invalid_api_key / quota_exceeded / openai_unavailable / deadline_exceeded
- **Telemetry (binding):** route_id, provider, model, duration_ms, input_tokens, output_tokens, outcome, cost_tag — INFO level. Pipeline layer logs, not adapter.
- **Test doubles:** httptest.NewServer fake handler — not interface mocks
- **Contract suite registration:** internal/adapter/contract/suite_test.go — OpenAI adapter must pass shared Provider contract suite
- **ESO pattern:** ExternalSecret → secretKeyRef — mirrors existing external-secret-vault-token.yaml
- **Two-repo scope for FR8–FR11:** Routing config entry lives in terminus-inference-provider-routing, not the gateway repo. Schema must be confirmed before writing FR8–FR11 stories (pre-check N3 from adversarial review).
- **FR22 (Art. 15 exception):** Already complete — docs/terminus/inference/openaiadapter/art15-exception.md committed.
- **Conditional registration in main.go:** OpenAI adapter only registered when GATEWAY_OPENAI_API_KEY is set. Absent key → adapter not registered → requests with openai/ prefix return 422 unsupported_model.

### FR Coverage Map

| FR | Epic | Summary |
|---|---|---|
| FR1 | Epic 1 | Gateway routes to OpenAI via registered adapter |
| FR2 | Epic 1 | ChatRequest → OpenAI HTTP mapping |
| FR3 | Epic 1 | OpenAI response → ChatResponse mapping |
| FR4 | Epic 1 | 401 → error envelope (unauthorized) |
| FR5 | Epic 1 | 429 → error envelope (rate_limited) |
| FR6 | Epic 1 | 5xx → error envelope (provider_error) |
| FR7 | Epic 1 | timeout → error envelope (timeout) |
| FR8 | Epic 3 | routing-config.yaml registration entry |
| FR9 | Epic 3 | Routing fallback selection |
| FR10 | Epic 3 | Restrict to workloadClass: interactive |
| FR11 | Epic 3 | degraded: true + fallback_provider: openai in response |
| FR12 | Epic 2 | API key stored in Vault |
| FR13 | Epic 2 | ESO ExternalSecret → GATEWAY_OPENAI_API_KEY |
| FR14 | Epic 2 | Key only from env var (constraint on Epic 1 code) |
| FR15 | Epic 2 | Helm chart — secretKeyRef in env: block |
| FR16 | Epic 1 | Startup log confirmation |
| FR17 | Epic 1 | Per-request structured telemetry |
| FR18 | Epic 1 | outcome field values |
| FR19 | Epic 1 | Contract test suite CI gate (Art. 2) |
| FR20 | Epic 4 | Operator smoke test against deployed gateway |
| FR21 | Epic 4 | Resilience test — degraded: true confirmed |
| FR22 | ✅ Pre-Complete | Art. 15 exception artifact committed |
| FR23 | Epic 3 | Batch prohibition at routing config layer |

## Epic List

### Epic 1: OpenAI Adapter Core

The inference gateway can accept requests, route them to OpenAI via a registered Provider-compliant adapter, return mapped responses, and emit structured telemetry — with contract tests passing as a CI gate.

**FRs covered:** FR1, FR2, FR3, FR4, FR5, FR6, FR7, FR16, FR17, FR18, FR19

**Repo:** terminus-inference-gateway

**Notes:** Adapter reads GATEWAY_OPENAI_API_KEY from env. Conditional registration: adapter only registers if env var is present. FR19 (contract suite) is a hard CI gate per Art. 2. FR16/FR17/FR18 are adapter-side behaviors co-located with implementation.

---

### Epic 2: Secure Key Delivery

Operators can provision the OpenAI API key in Vault and have it automatically delivered to the gateway pod as a Kubernetes env var via ESO — with no key material in source code, config files, or log output.

**FRs covered:** FR12, FR13, FR14, FR15

**Repo:** terminus-inference-gateway (Helm chart + ESO manifest)

**Notes:** Mirrors existing external-secret-vault-token.yaml pattern. secretKeyRef in env: block on deployment.yaml (NOT extraEnv). FR14 is a constraint verified against Epic 1 adapter code.

---

### Epic 3: Routing Policy

Operators can configure OpenAI as a priority-ordered fallback route in routing-config.yaml, restricted to interactive workloads, with batch requests prohibited — and the gateway reports fallback activation in its response.

**FRs covered:** FR8, FR9, FR10, FR11, FR23

**Repos:** terminus-inference-provider-routing (routing config) + terminus-inference-gateway (FR11 response decoration)

**Notes:** N3 resolved — routing config schema confirmed from provider-routing story 1.2: `schema_version: 1`, `providers:` list with `name/base_url/api_key_env/daily_budget_usd/price_per_1k_tokens`, `profiles:` list with `name/provider_chain`. Batch prohibition implemented by excluding openai from `batch` profile's provider_chain. **Dependencies:** provider-routing Epic 5 (stories 5.1–5.5 wiring Resolve into gateway pipeline) must be complete before Story 3.2.

---

### Epic 4: Acceptance Validation

Operators can run smoke and resilience tests against the deployed gateway to verify end-to-end OpenAI integration — confirming successful routing, fallback behavior, and degraded response signaling.

**FRs covered:** FR20, FR21

**Repo:** terminus-inference-gateway

**Notes:** Requires Epic 1 + 2 + 3 complete. FR20: smoke test via ESO-delivered key. FR21: Ollama unavailable → OpenAI selected → degraded: true confirmed.

---

## Stories

### Epic 1: OpenAI Adapter Core

---

#### Story 1.1: Add OpenAI error type constants and update error classification

As a **gateway developer**,
I want the error package to define `rate_limited`, `provider_error`, and `timeout` error type constants with matching HTTP status codes and adapter sentinels,
So that the OpenAI adapter can return strongly-typed errors that the pipeline maps to correct gateway responses (FR4, FR5, FR6, FR7).

**Acceptance Criteria:**

**Given** `internal/errors/errors.go` is extended with new exported constants
**When** the file is compiled
**Then** `ErrRateLimited = "rate_limited"`, `ErrProviderError = "provider_error"`, and `ErrTimeout = "timeout"` are present alongside existing constants
**And** `ErrAdapterUnauthorized` and `ErrAdapterRateLimited` sentinel `error` values are exported alongside existing `ErrAdapterUnavailable` and `ErrAdapterTimeout`

**Given** `StatusCode()` in `internal/errors/mapper.go` receives `ErrRateLimited`
**When** called
**Then** it returns `429`; `ErrProviderError` returns `503`; `ErrTimeout` returns `504`

**Given** `classifyAdapterError()` in `internal/pipeline/handler.go` receives an error wrapping `ErrAdapterUnauthorized`
**When** called
**Then** it returns `ErrUnauthorized`; an error wrapping `ErrAdapterRateLimited` returns `ErrRateLimited`

**Given** all existing tests run after these changes
**When** `make test` executes
**Then** all existing tests continue to pass — no regressions

**Technical Notes:**
- Target repo: `terminus-inference-gateway`
- Files: `internal/errors/errors.go`, `internal/errors/mapper.go`, `internal/pipeline/handler.go`
- Use `fmt.Errorf("openai: %w", ErrAdapterUnauthorized)` pattern in adapter; `errors.Is()` in classifier
- Do not remove or rename existing constants — Ollama error paths must be unaffected
- Update `internal/errors/errors_test.go` and `internal/pipeline/handler_test.go` for the new branches

---

#### Story 1.2: OpenAI adapter — full implementation with contract suite registration

As a **gateway user**,
I want to submit inference requests with model prefix `openai/` and receive mapped responses from the OpenAI `/v1/chat/completions` API,
So that the gateway can route requests to OpenAI as a registered provider (FR1, FR2, FR3, FR4, FR5, FR6, FR7, FR19).

**Acceptance Criteria:**

**Given** `New("test-key")` is called
**When** the constructor runs
**Then** it returns a non-nil `*OpenAIAdapter` and nil error
**And** `adapter.Name()` returns `"openai"`

**Given** `New("")` is called
**When** the constructor runs
**Then** it returns `(nil, error)` — empty API key is rejected at construction

**Given** a valid ChatRequest is submitted and the httptest server returns a well-formed OpenAI response
**When** `adapter.Chat(ctx, req)` is called
**Then** it returns a `*ChatResponse` with `Choices[0].Message.Content` set, `Usage.PromptTokens > 0`, `Usage.CompletionTokens > 0`
**And** the outbound HTTP request carries `Authorization: Bearer test-key` header
**And** the model name strips the `openai/` prefix before sending to OpenAI

**Given** the httptest server returns HTTP 401
**When** `adapter.Chat()` is called
**Then** it returns an error wrapping `ErrAdapterUnauthorized`

**Given** the httptest server returns HTTP 429
**When** `adapter.Chat()` is called
**Then** it returns an error wrapping `ErrAdapterRateLimited`

**Given** the httptest server returns HTTP 500 or 503
**When** `adapter.Chat()` is called
**Then** it returns an error wrapping `ErrAdapterUnavailable`

**Given** the request context deadline is exceeded before the server responds
**When** `adapter.Chat()` is called
**Then** it returns an error wrapping `ErrAdapterTimeout` (context.DeadlineExceeded chain)

**Given** the OpenAI adapter is registered in `contract/suite_test.go` using a happy-path httptest.NewServer
**When** `TestContractSuite` runs
**Then** the `OpenAIAdapter` case passes all suite assertions

**Technical Notes:**
- Target repo: `terminus-inference-gateway`
- New files: `internal/adapter/openai/adapter.go`, `internal/adapter/openai/types.go`, `internal/adapter/openai/adapter_test.go`
- Modified file: `internal/adapter/contract/suite_test.go` — add `newOpenAITestCase()` and append to `nonIntegrationAdapters`
- Constructor: `func New(apiKey string) (*OpenAIAdapter, error)` — no logger parameter (follows ollama pattern; telemetry is pipeline-layer responsibility per tech-decisions)
- Base URL: default `https://api.openai.com/v1`; overridable via `OPENAI_BASE_URL` env var (for test environments)
- Unexported mapper functions: `toOpenAIRequest(req *adapter.ChatRequest) openaiRequest` and `fromOpenAIResponse(resp openaiResponse) *adapter.ChatResponse`
- Unexported wire types: `openaiRequest`, `openaiResponse`, `openaiMessage`, `openaiChoice`, `openaiUsage` — OpenAI wire format must not leak outside the package
- `Health()` method: HTTP GET to `/v1/models` (or equivalent ping); return nil on 200, error otherwise
- All test cases are table-driven in `adapter_test.go`; httptest.NewServer used for all HTTP doubles
- API key injected as `"test-key"` in tests; httptest handler asserts `Authorization: Bearer test-key`
- Story depends on Story 1.1 (error sentinels must exist before adapter uses them)

---

#### Story 1.3: GatewayConfig extension and conditional OpenAI registration

As a **gateway operator**,
I want the gateway to register the OpenAI adapter at startup when `GATEWAY_OPENAI_API_KEY` is present and log a confirmation,
So that the adapter is available for routing without silent failure when the key is absent (FR1, FR16).

**Acceptance Criteria:**

**Given** `GATEWAY_OPENAI_API_KEY` is set to a non-empty value
**When** the gateway starts
**Then** the OpenAI adapter is registered in the registry with key `"openai"`
**And** an `slog.Info` line is emitted containing `"openai adapter registered"` (or equivalent confirmation message)

**Given** `GATEWAY_OPENAI_API_KEY` is empty or unset
**When** the gateway starts
**Then** the OpenAI adapter is NOT registered — no error, no crash
**And** a request with model `openai/gpt-4o-mini` returns HTTP 422 with `type: unsupported_model`

**Given** `GATEWAY_OPENAI_API_KEY` is set but `openai.New()` returns an error
**When** the gateway starts
**Then** the gateway exits with a fatal log — no silent degradation

**Given** `GatewayConfig` is loaded from environment
**When** `config.LoadConfig()` runs
**Then** `cfg.OpenAIAPIKey` is populated from `GATEWAY_OPENAI_API_KEY`
**And** `cfg.OpenAIBaseURL` is populated from `OPENAI_BASE_URL` (default `https://api.openai.com/v1`)
**And** `cfg.OpenAITimeoutSeconds` is populated from `OPENAI_TIMEOUT_SECONDS` (default `30`)

**Technical Notes:**
- Target repo: `terminus-inference-gateway`
- Modified files: `internal/config/config.go`, `internal/config/loader.go`, `internal/config/loader_test.go`, `cmd/gateway/main.go`
- Add `OpenAIAPIKey string`, `OpenAIBaseURL string`, `OpenAITimeoutSeconds int` to `GatewayConfig` struct
- Registration block in `main.go` (after ollama registration):
  ```go
  if cfg.OpenAIAPIKey != "" {
      openaiAdapter, err := openai.New(cfg.OpenAIAPIKey)
      if err != nil { log.Fatalf("openai adapter init failed: %v", err) }
      registry.Register(openaiAdapter)
      slog.Info("openai adapter registered", "model", cfg.OpenAIModel)
  }
  ```
- Story depends on Story 1.2 (openai package must exist)

---

#### Story 1.4: Pipeline telemetry enrichment for provider, tokens, and outcome

As a **gateway operator**,
I want every inference request to emit a structured log entry with provider, token counts, outcome, and cost tag,
So that I can observe cost and error patterns per provider (FR17, FR18).

**Acceptance Criteria:**

**Given** a successful inference request completes
**When** the pipeline emits the request log
**Then** the slog.InfoContext call includes fields: `provider` (adapter.Name()), `model`, `duration_ms` (total), `input_tokens` (Usage.PromptTokens), `output_tokens` (Usage.CompletionTokens), `outcome: "success"`, `cost_tag` (`"<provider>-interactive"` or `""` if unset)

**Given** an inference request fails with a rate_limited error
**When** the pipeline emits the log
**Then** `outcome` is `"rate_limited"`; `input_tokens` and `output_tokens` are `0`

**Given** an inference request fails with an unauthorized error
**When** the pipeline emits the log
**Then** `outcome` is `"unauthorized"`; `input_tokens` and `output_tokens` are `0`

**Given** an inference request fails with any other error
**When** the pipeline emits the log
**Then** `outcome` is `"error"`

**Given** the log entry is emitted at any log level
**When** inspected
**Then** no message content from `req.Messages` appears in any field — prompt content is never logged (NFR-D2)

**Technical Notes:**
- Target repo: `terminus-inference-gateway`
- Modified files: `internal/pipeline/handler.go`, `internal/pipeline/handler_test.go`
- Replace existing `slog.InfoContext(r.Context(), "request", "model", ..., "pre_adapter_ms", ..., "post_adapter_ms", ...)` with the full FR17 field set
- `outcome` derivation: on success: `"success"`; on error: call `classifyAdapterError(err)` and map type constant to outcome string (`ErrRateLimited` → `"rate_limited"`, `ErrUnauthorized` → `"unauthorized"`, default → `"error"`)
- `cost_tag` initial value: `provider.Name() + "-interactive"` (hardcoded for MVP; routing-aware cost tagging is Phase 2)
- Tokens are zero on any error path — `resp` will be nil
- `duration_ms` = elapsed time from request start to after adapter call (single field replacing pre/post split)
- Update `handler_test.go` to assert all new fields are present with correct values

---

### Epic 2: Secure Key Delivery

---

#### Story 2.1: ESO ExternalSecret for OpenAI API key

As a **gateway operator**,
I want an ESO ExternalSecret that syncs the OpenAI API key from Vault into the gateway namespace as a Kubernetes Secret,
So that the key is delivered securely without appearing in source code or config files (FR12, FR13).

**Acceptance Criteria:**

**Given** the ExternalSecret manifest is applied to the cluster
**When** ESO reconciles it
**Then** a Kubernetes Secret named `gateway-openai-api-key` exists in the `inference` namespace
**And** the secret contains key `GATEWAY_OPENAI_API_KEY` with the value from Vault path `secret/terminus/inference/gateway/openai-api-key`, field `gateway-openai-api-key`

**Given** the Vault path does not exist
**When** ESO attempts to sync
**Then** the ExternalSecret status reflects a sync error — no empty/null secret is created silently

**Given** the manifest is reviewed
**When** inspected
**Then** no API key value appears anywhere in the file — only the Vault path reference and secret key name

**Technical Notes:**
- Target repo: `terminus-inference-gateway`
- New file: `deploy/k8s/external-secret-openai-api-key.yaml`
- Mirrors `deploy/k8s/external-secret-vault-token.yaml` exactly in structure
- `metadata.name`: `gateway-openai-api-key`
- `metadata.namespace`: `inference`
- `argocd.argoproj.io/sync-wave: "2"` (same wave as vault-token)
- `spec.secretStoreRef.name`: `vault-backend`
- `spec.target.name`: `gateway-openai-api-key`
- `spec.data[0].secretKey`: `GATEWAY_OPENAI_API_KEY`
- `spec.data[0].remoteRef.key`: `secret/terminus/inference/gateway/openai-api-key`
- `spec.data[0].remoteRef.property`: `gateway-openai-api-key`
- Update `deploy/k8s/kustomization.yaml` to include the new manifest

---

#### Story 2.2: Helm chart conditional env var for OpenAI API key

As a **gateway operator**,
I want the gateway Helm chart to pass `GATEWAY_OPENAI_API_KEY` into the pod environment from the ESO-synced secret when configured,
So that the adapter can read the key at startup without it appearing in values files (FR14, FR15).

**Acceptance Criteria:**

**Given** `values.yaml` has `openai.keySecret: "gateway-openai-api-key"` set
**When** the Helm chart renders
**Then** `deployment.yaml` contains an `env:` entry for `GATEWAY_OPENAI_API_KEY` using `secretKeyRef` with `name: gateway-openai-api-key` and `key: GATEWAY_OPENAI_API_KEY`

**Given** `values.yaml` has `openai.keySecret: ""` (default)
**When** the Helm chart renders
**Then** no `GATEWAY_OPENAI_API_KEY` env entry appears in the rendered deployment — adapter not registered (FR14 constraint)

**Given** the rendered deployment manifest is inspected
**When** reviewed for secrets
**Then** the API key value never appears — only the secretKeyRef reference is present

**Technical Notes:**
- Target repo: `terminus-inference-gateway`
- Modified files: `deploy/helm/templates/deployment.yaml`, `deploy/helm/values.yaml`
- Pattern mirrors existing `GATEWAY_VAULT_TOKEN` env entry but wrapped in conditional:
  ```yaml
  {{- if .Values.openai.keySecret }}
  - name: GATEWAY_OPENAI_API_KEY
    valueFrom:
      secretKeyRef:
        name: {{ .Values.openai.keySecret }}
        key: GATEWAY_OPENAI_API_KEY
  {{- end }}
  ```
- Add to `values.yaml`:
  ```yaml
  openai:
    keySecret: ""          # set to ESO secret name to enable OpenAI adapter
    model: gpt-4o-mini     # default model
    timeoutSeconds: 30
  ```
- Do NOT use `extraEnv` — confirmed incorrect from `deploy/helm/templates/deployment.yaml` source

---

### Epic 3: Routing Policy

**Epic Pre-condition:** Stories 3.2 depends on `terminus-inference-provider-routing` Epic 5 stories (5.1–5.5) being complete — those stories wire the routing library's `Resolve()` and `RecordUsage()` calls into the gateway pipeline. Story 3.1 is independent and can be done first.

---

#### Story 3.1: Add OpenAI provider entry to routing configuration

As a **gateway operator**,
I want to add OpenAI as a registered provider in `routing-config.yaml` with priority, budget, and workload class restrictions,
So that the routing engine can select OpenAI as a fallback for interactive workloads while prohibiting batch routing to OpenAI (FR8, FR9, FR10, FR23).

**Acceptance Criteria:**

**Given** `routing-config.yaml` is updated and the gateway reloads config
**When** `LoadProfiles()` parses the file
**Then** an `openai` provider entry exists with `base_url: https://api.openai.com/v1`, `api_key_env: GATEWAY_OPENAI_API_KEY`, `daily_budget_usd: 5.00`, `price_per_1k_tokens: 0.002`

**Given** a request arrives with workload class `interactive` and Ollama is at capacity
**When** the routing engine's fallback logic runs
**Then** `openai` is selected as the second entry in the `interactive` profile's `provider_chain`

**Given** a request arrives with workload class `batch`
**When** the routing engine evaluates the `batch` profile
**Then** `openai` does NOT appear in the `batch` profile's `provider_chain` — OpenAI is never selected for batch (FR23)

**Given** the routing config is reviewed
**When** inspected
**Then** `schema_version: 1` is present and the OpenAI entry passes `LoadProfiles` validation without error

**Technical Notes:**
- Target repo: `terminus-inference-provider-routing` (routing-config.yaml in Helm ConfigMap or config directory from story 5.4)
- Routing config schema (confirmed from provider-routing story 1.2):
  ```yaml
  schema_version: 1
  providers:
    - name: ollama
      base_url: http://ollama.inference.svc.cluster.local:11434
      api_key_env: ""
      daily_budget_usd: 0
      price_per_1k_tokens: 0
    - name: openai
      base_url: https://api.openai.com/v1
      api_key_env: GATEWAY_OPENAI_API_KEY
      daily_budget_usd: 5.00
      price_per_1k_tokens: 0.002
  profiles:
    - name: interactive
      provider_chain: [ollama, openai]
    - name: batch
      provider_chain: [ollama]
  ```
- Route identifier (Art. 1 profile): `openai-fallback`, model target: `gpt-4o-mini`, allowed workload classes: `[interactive]`
- `daily_budget_usd: 5.00` is the initial operator-configured budget ceiling (FR8)
- Confirm exact config file path from provider-routing story 5.4 deployment artifacts before implementing

---

#### Story 3.2: Routing integration validation and degraded response wiring

As a **gateway user**,
I want requests that fall back to OpenAI to include `degraded: true` and `fallback_provider: openai` in the response,
So that callers know which provider handled their request and can react accordingly (FR9, FR10, FR11).

**Acceptance Criteria:**

**Given** the gateway pipeline calls `router.Resolve()` (provider-routing Epic 5 pre-condition) and Ollama is at capacity
**When** a request with `X-Workload-Class: interactive` arrives
**Then** `Resolve` returns `openai` as the selected provider with `Degraded: true`
**And** the gateway dispatches to the OpenAI adapter
**And** the response body includes `degraded: true` and `fallback_provider: "openai"`

**Given** a request arrives with `X-Workload-Class: batch`
**When** routing is evaluated
**Then** the routing engine never selects OpenAI (`openai` not in batch profile)
**And** if Ollama is also unavailable, the gateway returns 503 with `type: provider_unavailable`

**Given** OpenAI is selected as the primary provider (not fallback) for an interactive request
**When** the response is returned
**Then** `degraded` is absent or `false` — degraded flag only set on non-primary selections

**Technical Notes:**
- Target repo: `terminus-inference-gateway`
- **Pre-condition:** Provider-routing initiative Epic 5 stories (5.1–5.5) must be complete — `router.Resolve()` must be wired into `internal/pipeline/handler.go` before this story can be validated
- The `RoutingResponse.Degraded` and `RoutingResponse.FallbackProvider` fields are propagated to the API response by story 5.2 of provider-routing — this story validates that the OpenAI routing config entry correctly triggers those fields
- Acceptance validation: integration test or local deployment test with Ollama stopped, routing config loaded, OpenAI adapter registered with test key pointing at httptest server
- No new gateway code required if provider-routing Epic 5 is complete — this story is an integration validation story

---

### Epic 4: Acceptance Validation

**Epic Pre-condition:** Epics 1, 2, and 3 must be complete and deployed to the k3s cluster with ESO key sync active.

---

#### Story 4.1: Operator smoke test — live OpenAI response from deployed gateway

As a **gateway operator**,
I want to run a smoke test that produces a valid non-error response from the OpenAI adapter via the deployed gateway using the ESO-delivered key,
So that I can confirm the full integration stack is wired correctly in the deployed environment (FR20).

**Acceptance Criteria:**

**Given** the gateway is running on k3s with ESO-synced `GATEWAY_OPENAI_API_KEY` populated
**When** a POST to `/v1/chat/completions` is made with `model: openai/gpt-4o-mini` and valid messages payload
**Then** the response is HTTP 200 with a non-empty `choices[0].message.content`
**And** no test credential or prompt content appears in gateway logs (NFR-D2)

**Given** the smoke test script or runbook is committed
**When** reviewed
**Then** it contains no hardcoded API key values — key is sourced from the deployed environment only

**Given** the deployed gateway logs are inspected after the smoke test
**When** reviewed
**Then** an INFO log entry exists with `provider: openai`, `outcome: success`, `input_tokens > 0`, `output_tokens > 0`

**Technical Notes:**
- Target repo: `terminus-inference-gateway`
- A smoke test script or Makefile target (e.g., `make smoke-openai`) that:
  1. Uses `kubectl port-forward` to reach the gateway
  2. Reads the Vault token from environment (not hardcoded)
  3. Sends a minimal valid request
  4. Asserts HTTP 200 and non-empty content
- Alternatively: a short runbook in `docs/` with manual steps if scripted test is not feasible in environment
- No real API key in any committed file

---

#### Story 4.2: Resilience test — Ollama unavailable routes to OpenAI with degraded signal

As a **gateway operator**,
I want to verify that when Ollama is unavailable the routing engine selects OpenAI and the response includes `degraded: true`,
So that I have confirmed evidence of the fallback path operating as designed (FR21).

**Acceptance Criteria:**

**Given** Ollama is stopped or unreachable in the deployed environment
**When** a POST to `/v1/chat/completions` with `X-Workload-Class: interactive` and `model: openai/gpt-4o-mini` is submitted
**Then** the gateway returns HTTP 200
**And** the response body contains `"degraded": true` and `"fallback_provider": "openai"`

**Given** the resilience test is complete and Ollama is restarted
**When** a subsequent request is submitted
**Then** Ollama is selected again (not OpenAI) with no degraded flag

**Given** the gateway logs are inspected during the resilience test
**When** reviewed
**Then** a log entry shows `provider: openai`, `outcome: success`, and routing degradation was triggered

**Technical Notes:**
- Target repo: `terminus-inference-gateway`
- Method: stop the Ollama deployment (`kubectl scale deployment ollama --replicas=0`) during test; restore after
- Test may be manual runbook or a Makefile target
- Depends on routing-config.yaml having `provider_chain: [ollama, openai]` for the `interactive` profile (Story 3.1 complete)
- Depends on `degraded: true` propagation being wired (Story 3.2 complete)
