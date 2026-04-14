---
initiative: terminus-inference-openaiadapter
phase: techplan
author: electricm0nk
date: 2026-04-07
status: complete
---

# Technical Decisions — terminus-inference-openaiadapter

Lean decisions log for agent and developer reference. Full rationale in `architecture.md`.

---

## Language & Runtime

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Language | Go (latest stable) | Extension of the existing gateway — not a new binary. Same interface, same compile-time contract enforcement. |
| Deployment target | k3s (terminus-infra-k3s) | Existing gateway Helm chart, co-located in gateway repo. No new chart required. |
| Module location | `internal/adapter/openai/` | Follows existing `internal/adapter/ollama/` structure. |

---

## HTTP Client (Outbound to OpenAI)

| Decision | Choice | Rationale |
|----------|--------|-----------|
| HTTP client | `net/http` stdlib | Retry is a routing-layer concern (provider-routing initiative). The adapter makes one attempt and returns a structured error on failure. No retryablehttp dependency introduced. |
| Timeout enforcement | Request context deadline | Context from `Chat(ctx, req)` carries the deadline set by Helm `openai.timeoutSeconds`. No secondary timeout inside the adapter — no deadline circumvention possible. |
| OpenAI API version | `/v1/chat/completions` pinned | Breaking API change requires a new initiative. Version lock is explicit and non-negotiable. |
| Target endpoint | `https://api.openai.com/v1/chat/completions` | Hardcoded default; overridable via `OPENAI_BASE_URL` env var for test environments. |

---

## Authentication & Key Delivery

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Key injection | `secretKeyRef` in `env:` block | Consistent with existing gateway pattern (`VaultToken`). NOT `extraEnv` — confirmed from `deploy/helm/templates/deployment.yaml`. |
| Secret source | ExternalSecret (ESO) → Vault | `external-secret-openai-api-key.yaml` mirrors `external-secret-vault-token.yaml`. Vault is the canonical secret store (constitutional constraint). |
| Key read timing | Once at construction (`New()`) | Key read from env var at `New(apiKey string)` call. Empty key == validation error at startup, not at first call. |
| Auth header | `Authorization: Bearer <api-key>` | Standard OpenAI auth. Never logged. |

---

## Adapter Registration

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Registry key | `"openai"` (via `Name()`) | Registry is fully data-driven — key comes from `adapter.Name()`, never hardcoded at the registry level. Model strings take the form `openai/gpt-4o-mini`. |
| Conditional registration | Only when `OPENAI_API_KEY` is set | `main.go` guards registration: if key absent, adapter is not registered. Requests with `openai/` prefix return 422 `unsupported_model` — no silent failure. |
| Default model | `gpt-4o-mini` | Configured via Helm `providers.openai.model`. Balances cost and capability for default inference workloads. Operator-overridable. |
| Config fields added | `OpenAIAPIKey`, `OpenAIModel`, `OpenAITimeoutSeconds` | Added to `GatewayConfig` alongside existing `OllamaBaseURL` / `OllamaTimeoutSeconds` fields. |

---

## Logging & Telemetry

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Logger | Stdlib `log/slog` | Consistent with `internal/pipeline/handler.go` and `internal/auth/middleware.go`. No logger injection into the adapter — confirmed from ollama reference implementation (zero logging in adapter). |
| Telemetry responsibility | Pipeline layer owns logging | Adapter returns `*ChatResponse` with `Usage` populated. Pipeline calls `slog.InfoContext` with telemetry fields. Adapter does not log. |
| Telemetry fields | `route_id, provider, model, duration_ms, input_tokens, output_tokens, outcome, cost_tag` | INFO level. Fields emitted by pipeline after `Chat()` returns. `Usage` on `ChatResponse` carries token counts. |
| Sensitive data | API key never appears in logs | Adapter and pipeline log no credential values — enforced by construction (key stored in struct field, never passed through logging calls). |

---

## Error Handling

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Error enum | `unauthorized` / `rate_limited` / `provider_error` / `timeout` | Consistent with gateway error taxonomy. All adapter errors map to one of these four before returning to the pipeline. |
| Error wrapping | `fmt.Errorf("openai: %w", err)` | Mirrors ollama pattern — provider internals never reach HTTP callers. |
| OpenAI status mapping | 401 → `unauthorized`, 429 → `rate_limited`, 5xx → `provider_error`, deadline exceeded → `timeout` | Deterministic. No ambiguous `other` category. |
| Error detail forwarding | Never | OpenAI error body details are not forwarded to gateway callers (NFR-S3). |

---

## Mapper Strategy

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Mapper functions | Unexported `toOpenAIRequest()` / `fromOpenAIResponse()` | Encapsulated inside `internal/adapter/openai/` package. Not part of the `Provider` interface. Tested via table-driven `adapter_test.go`. |
| Internal types | `openaiRequest` / `openaiResponse` (unexported structs) | OpenAI wire format does not leak outside the package. Only `adapter.ChatRequest` / `adapter.ChatResponse` cross the boundary. |
| System prompt | Prepended as `role: system` message | If `req.SystemPrompt` is non-empty, it is prepended to the messages slice. No gateway-level system prompt injection logic needed. |

---

## Testing

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Test doubles | `httptest.NewServer` fake handler | Mirrors the outbound HTTP boundary precisely. No interface mocks — the real `net/http` client exercises the real marshal/unmarshal/error path. More resilient than mock-based stubs. |
| Unit test placement | `internal/adapter/openai/adapter_test.go` | Co-located, table-driven. Follows gateway-wide convention. |
| Contract suite registration | `internal/adapter/contract/suite_test.go` | OpenAI adapter must pass the shared `Provider` contract suite. A fake `httptest.NewServer` configured for success is registered under `"openai"`. |
| API key in tests | Injected constant `"test-key"` via `New()` | No real credentials in tests. `httptest.NewServer` validates the `Authorization` header value. |

---

## Deferred Decisions

| Area | Status | Path Forward |
|------|--------|-------------|
| Streaming responses | Out of scope (MVP) | Pipeline currently rejects streaming before adapter is called. No adapter-level change needed. |
| Model aliasing | Deferred | If `openai/gpt-4o` needs to map to `gpt-4o-mini` under cost caps, add a model resolver in the routing layer — not in this adapter. |
| Response caching | Out of scope | Routing/infrastructure concern. No adapter-level state. |
| Art. 15 exception document | Parallel story | `docs/terminus/inference/openaiadapter/art15-exception.md` must be authored and committed before medium promotion gate. Not a code story. |
