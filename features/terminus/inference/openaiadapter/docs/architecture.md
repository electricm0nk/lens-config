---
stepsCompleted: [step-01-init, step-02-context, step-03-starter, step-04-decisions, step-05-patterns, step-06-structure, step-07-validation, step-08-complete]
lastStep: 8
status: complete
completedAt: '2026-04-07'
inputDocuments:
  - docs/terminus/inference/openaiadapter/prd.md
  - docs/terminus/inference/gateway/prd.md
  - docs/terminus/inference/gateway/architecture.md
  - docs/terminus/inference/gateway/tech-decisions.md
  - docs/terminus/inference/provider-routing/prd.md
workflowType: architecture
project_name: terminus-inference-openaiadapter
user_name: electricm0nk
date: '2026-04-07'
---

# Architecture Decision Document — terminus-inference-openaiadapter

_This document builds collaboratively through step-by-step discovery. Sections are appended as we work through each architectural decision together._

---

## Project Context Analysis

### Requirements Overview

**Functional Requirements (23 across 7 areas):**
- Provider Adapter (FR1–7): `OpenAIAdapter` implementing `ProviderAdapter` interface; maps internal request/response structs to OpenAI `/v1/chat/completions`; maps 401→`unauthorized`, 429→`rate_limited`, 5xx→`provider_error`, timeout→`timeout`
- Routing Configuration (FR8–11): OpenAI entry in `routing-config.yaml`; fallback selection when Ollama unavailable; `allowedClasses: [interactive]` enforcement; `degraded: true` propagation
- Secrets & Key Delivery (FR12–14): Vault secret, ESO ExternalSecret, env var runtime read — key never persisted post-request
- Deployment & Wiring (FR15–16): Helm `extraEnv` passthrough; startup registration log
- Observability & Telemetry (FR17–18): Structured INFO log per request (`route_id`, `provider`, `model`, `duration_ms`, `input_tokens`, `output_tokens`, `outcome`, `cost_tag`)
- Testing & Validation (FR19–21): Contract test gate, smoke test, resilience test
- Compliance (FR22–23): Art. 15 exception artifact; batch workload exclusion

**Non-Functional Requirements (11):**
- Performance: ≤50ms adapter overhead; no post-request key retention
- Security: Key never logged; TLS only; annual Art. 15 renewal
- Reliability: No retry inside adapter (routing layer owns it); no response caching
- Observability: Telemetry at INFO level
- Data Governance: Interactive workloads only; no prompt content logging

### Technical Constraints & Dependencies

- `ProviderAdapter` Go interface is fixed — adapter must conform exactly
- Structured error envelope `{type, provider, reason, message}` is fixed contract (gateway Art. 2)
- ESO + Vault secrets pattern is the only permissible key delivery path (NFR-S2)
- `routing-config.yaml` schema must accommodate an OpenAI entry with `allowedClasses` field — confirm existing schema supports this before implementation
- Inference service constitution Art. 1 requires a complete route profile before the route is deployed

### Cross-Cutting Concerns

| Concern | Scope | Mechanism |
|---------|-------|-----------|
| API key hygiene | Adapter + Helm + CI | Env var only; never logged; no config file entry |
| Art. 7 data sovereignty | Routing config | `allowedClasses: [interactive]` in route entry |
| Telemetry | Adapter impl | Structured INFO log on every `Complete()` call |
| Art. 15 AI safety exception | Planning artifact | 7-item document committed before medium promotion |
| Inference Art. 1 route profile | Architecture artifact | Route profile fields declared in this document |
| Contract test gate | CI | Adapter must be covered before promotion |

---

## Starter Template Evaluation

### Primary Technology Domain

API backend — Go in-process adapter, extending an existing deployed service (brownfield).

### No External Starter Applicable

This initiative extends the `terminus-inference-gateway` repository, which is already bootstrapped and deployed. There is no greenfield project scaffold required. All language, tooling, testing, and deployment foundations are inherited from the existing gateway codebase.

### Inherited Foundation

| Element | Source | Notes |
|---------|--------|-------|
| Go module structure | `terminus-inference-gateway` | No change |
| `ProviderAdapter` interface | Gateway codebase | Adapter must implement exactly |
| `CompletionRequest` / `CompletionResponse` | Gateway codebase | Passed through without modification |
| Structured error envelope | Gateway codebase | `{type, provider, reason, message}` |
| Test suite infrastructure | Gateway test suite | Extend with `OpenAIAdapter` coverage |
| Helm chart | Gateway Helm chart | Add `extraEnv` for `GATEWAY_OPENAI_API_KEY` |
| `routing-config.yaml` | Gateway config repo | Add OpenAI provider entry |

**First implementation story:** Confirm `ProviderAdapter` interface signature from gateway source before writing any adapter code — ensures zero-drift compliance.

---

## Core Architectural Decisions

### Decision Priority Analysis

**Critical (Block Implementation):**
- HTTP client strategy (determines adapter structure)
- Key delivery and storage (determines startup contract)
- Test double strategy (determines what contract tests can cover)

**Important (Shape Architecture):**
- Request/response mapping approach (determines testability)
- Telemetry emission pattern (determines logger dependency)

**Deferred (Post-MVP):**
- Per-model routing within the adapter (Phase 2)
- Cost alerting hooks (Phase 2)

### API & Communication

**HTTP Client:** `net/http` stdlib — no retry logic inside the adapter. Retry and fallback decisions belong to the routing engine. The adapter is responsible only for a single attempt and clean error mapping.

**Request/Response Mapping:** Separate `toOpenAIRequest()` and `fromOpenAIResponse()` mapper functions within the adapter package. These are independently testable and keep `Complete()` focused on the call path only.

**OpenAI API Target:** `POST https://api.openai.com/v1/chat/completions` — OpenAI v1.

### Authentication & Security

**Key Storage:** `GATEWAY_OPENAI_API_KEY` is read once at adapter construction via `os.Getenv()` and stored as an unexported struct field. Key is validated (non-empty check) at construction time — startup fails fast if missing. Key is not re-read per request and is never logged.

**TLS:** All outbound calls use `https://` — standard `net/http` client enforces TLS by default (NFR-S3).

### Observability

**Telemetry:** The existing gateway structured logger is injected into the adapter via constructor. Every `Complete()` call emits one INFO-level structured log entry at completion (success or error), with fields: `route_id`, `provider`, `model`, `duration_ms`, `input_tokens`, `output_tokens`, `outcome`, `cost_tag`. No debug-level telemetry required.

### Infrastructure & Deployment

**Key Delivery Sequence:** ESO `ExternalSecret` sync must complete before the gateway pod starts. If `GATEWAY_OPENAI_API_KEY` env var is missing at startup, the adapter constructor returns an error and the gateway boot fails — misconfigured deployments are caught at start, not during the first live request.

**Helm Wiring:** `extraEnv` entry in gateway `values.yaml`, following the existing pattern confirmed from prior gateway deployment (PRD Journey 3).

### Testing Strategy

**Contract Tests:** `httptest.NewServer` fake handler implements the OpenAI `/v1/chat/completions` response shape. Tests cover: happy path, 401, 429, 5xx, timeout. This exercises the actual HTTP call path and mapper functions end-to-end.

**Unit Tests:** Mapper functions (`toOpenAIRequest`, `fromOpenAIResponse`) tested directly with known inputs/outputs.

**Integration/Smoke Test:** Deployed-environment test using ESO-delivered key against live OpenAI API (PRD Journey 4).

### Inference Art. 1 — Route Profile

Per inference service constitution Article 1, a complete route profile must exist before the route is deployed:

| Field | Value |
|-------|-------|
| Route identifier | `openai-fallback` |
| Model target | `gpt-4o-mini` (configurable via `routing-config.yaml`) |
| Request timeout | 30s |
| Retry policy | None (routing engine owns retry) |
| Escalation on exhausted retries | Structured error → routing engine marks provider unavailable |
| Cost tag | `openai-interactive` |
| Allowed workload classes | `interactive` only |

### Decision Impact Analysis

**Implementation Sequence:**
1. Confirm `ProviderAdapter` interface signature from gateway source
2. Implement `OpenAIAdapter` struct with constructor (key validation at boot)
3. Implement mapper functions (`toOpenAIRequest`, `fromOpenAIResponse`)
4. Implement `Complete()` with telemetry emission
5. Extend contract test suite with `httptest` fake handler
6. Add OpenAI entry to `routing-config.yaml`
7. Add ESO `ExternalSecret` + Helm `extraEnv`
8. Smoke test + resilience test in deployed environment

**Cross-Component Dependencies:**
- Adapter startup depends on ESO sync completing before pod launch
- Contract tests depend on mapper correctness
- Routing engine's `degraded: true` and `allowedClasses` enforcement are routing-config-owned — not adapter-owned

---

## Implementation Patterns & Consistency Rules

### Conflict Points Identified

6 areas where implementation agents could make incompatible choices without explicit guidance.

### Error Type Enum

All error type values are lowercase snake_case strings matching the gateway's existing structured error envelope. Agents must use exactly these values:

| Condition | `type` value | `reason` value |
|-----------|-------------|----------------|
| OpenAI 401 | `"unauthorized"` | `"invalid_api_key"` |
| OpenAI 429 | `"rate_limited"` | `"quota_exceeded"` |
| OpenAI 5xx | `"provider_error"` | `"openai_unavailable"` |
| Request timeout | `"timeout"` | `"deadline_exceeded"` |

### Telemetry Log Field Naming

Structured log fields must use these exact names (snake_case only — no camelCase alternatives):

```
route_id, provider, model, duration_ms, input_tokens, output_tokens, outcome, cost_tag
```

`outcome` values: `"success"`, `"error"`, `"rate_limited"`, `"unauthorized"`. Must be set before the log call.

### Constructor Signature Pattern

```go
func NewOpenAIAdapter(apiKey string, logger Logger) (*OpenAIAdapter, error)
```

- `apiKey` validated non-empty at construction; constructor returns `error` if empty
- `logger` is the gateway's existing structured logger interface — injected, not instantiated inside adapter
- No exported fields on the struct

### Mapper Function Naming

```go
func toOpenAIRequest(req CompletionRequest) openAIRequest       // unexported
func fromOpenAIResponse(resp openAIResponse) CompletionResponse // unexported
```

Both unexported — package-internal only.

### routing-config.yaml Field Conventions

Confirm field names (`allowedClasses` vs `allowed_classes`) from the existing config file before writing the OpenAI entry. Do not introduce new field names.

### Test File Placement

Go conventions: `openai_adapter_test.go` co-located with `openai_adapter.go`. No separate `tests/` directory.

### All Implementation Agents MUST

- Use error type strings from the enum table above — no freeform error types
- Use exact log field names from the telemetry field list — no renaming
- Inject logger via constructor — never instantiate a new logger inside the adapter
- Never store the API key in a variable named anything other than `apiKey` (unexported field)
- Return `error` from constructor if API key is empty — no silent degradation
- Use `httptest.NewServer` for HTTP test doubles — not interface mocks for HTTP calls

---

## Project Structure & Boundaries

### Repository: terminus-inference-gateway

This initiative adds one new package to an existing repository. No structural changes to existing packages.

```
terminus-inference-gateway/
├── internal/
│   └── adapter/
│       ├── provider.go                          # EXISTING — Provider interface, ChatRequest, ChatResponse
│       ├── registry.go                          # EXISTING — adapter registry
│       ├── ollama/
│       │   ├── adapter.go                       # EXISTING — reference implementation
│       │   ├── adapter_test.go
│       │   └── types.go
│       ├── openai/                              # NEW PACKAGE
│       │   ├── adapter.go                       # OpenAI Provider implementation
│       │   ├── adapter_test.go                  # Unit tests (mapper functions)
│       │   ├── adapter_integration_test.go      # httptest.NewServer contract tests
│       │   └── types.go                         # OpenAI wire format structs
│       └── contract/
│           └── suite_test.go                    # MODIFIED — register OpenAI adapter
├── deploy/
│   ├── helm/
│   │   ├── values.yaml                          # MODIFIED — add providers.openai stanza
│   │   └── templates/
│   │       └── deployment.yaml                  # MODIFIED — add GATEWAY_OPENAI_API_KEY secretKeyRef
│   └── k8s/
│       └── external-secret-openai-api-key.yaml  # NEW — ESO ExternalSecret
```

### Interface Ground Truth (Corrected from PRD)

The actual `Provider` interface (`internal/adapter/provider.go`):

```go
type Provider interface {
    Chat(ctx context.Context, req *ChatRequest) (*ChatResponse, error)
    Health(ctx context.Context) error
    Name() string
}
```

**Struct names:** `ChatRequest`, `ChatResponse`, `Usage`, `Choice`, `Message` — **not** `CompletionRequest`/`CompletionResponse` as assumed in PRD. This document is authoritative.

### Constructor Pattern

```go
// internal/adapter/openai/adapter.go
func New(apiKey string) (*OpenAIAdapter, error)
```

Matches the `ollama.New(baseURL)` pattern. Key validated non-empty at construction; `error` returned if missing. Logger pattern: confirm from gateway codebase whether telemetry uses `slog` or a gateway logger package before implementation — must match existing adapter telemetry approach.

### Helm Wiring Pattern

The deployment uses `secretKeyRef` in `env:` (not `extraEnv`). Add to `values.yaml`:

```yaml
providers:
  openai:
    enabled: false         # off by default; operator enables
    apiKeySecret: ""       # k8s Secret name containing GATEWAY_OPENAI_API_KEY
    model: "gpt-4o-mini"
    timeoutSeconds: 30
```

Add to `deploy/helm/templates/deployment.yaml` `env:` block:

```yaml
{{- if .Values.providers.openai.enabled }}
- name: GATEWAY_OPENAI_API_KEY
  valueFrom:
    secretKeyRef:
      name: {{ required "providers.openai.apiKeySecret required when openai enabled" .Values.providers.openai.apiKeySecret }}
      key: GATEWAY_OPENAI_API_KEY
{{- end }}
```

### ESO ExternalSecret Pattern

`deploy/k8s/external-secret-openai-api-key.yaml` follows `external-secret-vault-token.yaml` shape:

```yaml
spec:
  data:
    - secretKey: GATEWAY_OPENAI_API_KEY
      remoteRef:
        key: secret/terminus/inference/gateway/openai-api-key
        property: api-key
```

### Routing Config Entry (Provider-Routing Repo)

The OpenAI provider entry lives in the provider-routing config (separate initiative). Required entry shape:

```yaml
providers:
  - name: openai-fallback
    adapter: openai
    model: gpt-4o-mini
    priority: 2
    allowedClasses: [interactive]
    budgetCeiling: 10.00
    timeoutSeconds: 30
    costTag: openai-interactive
```

### Architectural Boundaries

| Boundary | Owner | Notes |
|----------|-------|-------|
| HTTP call to OpenAI API | `internal/adapter/openai` | Single attempt, clean error map |
| Retry / fallback logic | Provider-routing engine | Out of scope for this adapter |
| API key rotation | Vault + ESO | Adapter reads env var at startup only |
| Workload class enforcement | `routing-config.yaml` | Not enforced inside adapter |
| `degraded: true` flag | Routing engine | Not set by adapter |

---

## Architecture Validation

### Coherence Validation ✅

All decisions are compatible. Technology choices (`net/http`, `httptest.NewServer`, `secretKeyRef`) are fully aligned with existing gateway infrastructure patterns. One implementation pre-check required: confirm logger package/approach from gateway source before writing telemetry calls — must match `ollama` adapter pattern.

### Requirements Coverage

**All 23 FRs architecturally supported.** One gap resolved during validation:

- **FR22 / Art. 15 exception artifact path:** artifact committed to `docs/terminus/inference/openaiadapter/art15-exception.md` in the control repo before medium promotion. Not in the gateway source repo.

**All 11 NFRs addressed.** Key mappings:
- NFR-S1/S2 (key hygiene): constructor validation + `secretKeyRef` pattern
- NFR-S3 (TLS): `https://` enforced by `net/http` default
- NFR-R1 (no retry in adapter): explicit in HTTP client decision
- NFR-D2 (no prompt logging): explicit in patterns section

**Constitutional compliance:**
- Inference Art. 1: route profile declared ✅
- Inference Art. 2: contract suite extension specified ✅
- Art. 7: `allowedClasses: [interactive]` in routing config ✅
- Art. 10: INFO-level structured telemetry per request ✅
- Art. 15: exception artifact path resolved — `docs/terminus/inference/openaiadapter/art15-exception.md` ✅

### Architecture Completeness Checklist

- [x] Project context analyzed and constraints mapped
- [x] Brownfield foundation confirmed from source — interface names corrected from PRD assumptions
- [x] All architectural decisions documented with rationale
- [x] Implementation patterns defined with exact naming rules
- [x] Project structure mapped with new and modified files identified
- [x] Helm and ESO patterns confirmed from existing infrastructure
- [x] Route profile declared (Inference Art. 1)
- [x] Art. 15 artifact path resolved
- [x] All 23 FRs and 11 NFRs traced to architectural elements

### Architecture Readiness Assessment

**Overall Status: READY FOR IMPLEMENTATION**

**Key Strengths:**
- Interface and struct names verified from source — zero drift risk
- Helm and ESO patterns lifted exactly from existing infrastructure
- Error type enum and telemetry field names fully specified — no agent ambiguity
- Routing config boundary is explicit — adapter scope is tightly bounded

**Implementation Pre-Checks (before first story):**
1. Confirm logger package name from gateway source (used in `Chat()` telemetry call)
2. Confirm `routing-config.yaml` field names from provider-routing repo when available
3. Confirm `contract/suite_test.go` adapter registration pattern from existing test

### Implementation Handoff

**First story:** Create `internal/adapter/openai/adapter.go` — implement `New()`, `Name()`, `Health()`, `Chat()` against the `Provider` interface. Mapper functions in same file or `types.go`. All error types and telemetry fields from patterns section are binding.

**Art. 15 exception artifact:** Author `docs/terminus/inference/openaiadapter/art15-exception.md` as a parallel story — must be committed before medium promotion gate.
