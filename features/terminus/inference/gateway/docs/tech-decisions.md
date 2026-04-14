---
initiative: terminus-inference-gateway
phase: techplan
author: electricm0nk
date: 2026-04-03
status: complete
---

# Technical Decisions — terminus-inference-gateway

Lean decisions log for agent and developer reference. Full rationale in `architecture.md`.

---

## Language & Runtime

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Language | Go (latest stable) | Adapter pattern maps directly onto Go interfaces — compile-time enforced, no framework overhead. Single static binary on k3s. Significant NFR-P1 headroom (~0.2ms baseline vs 100ms budget). |
| Runtime target | k3s (terminus-infra-k3s) | Deployment substrate is known; Helm chart co-located in gateway repo. |

## HTTP Layer

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Router | `github.com/go-chi/chi/v5` | Lightweight, idiomatic, middleware-composable. Maps cleanly to the OpenAI-compatible route surface. |
| API spec | OpenAI API v1 (locked) | Non-negotiable constraint. Incompatibility is a breaking change requiring a new initiative. |
| Spec tooling | `oapi-codegen` from `api/openapi.yaml` | Spec is source-of-truth. Generated types and `ServerInterface` stubs flow from it; all contract tests target generated types. |

## Authentication

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Mechanism | `Authorization: Bearer <vault-token>` | Primary consumers call from inside k8s with Vault-issued tokens. Covers dev/operator access from outside the cluster. |
| Validation | Vault `/v1/auth/token/lookup-self` per request | No session caching (NFR-S4). Auth on every request. |
| Startup gate | Vault unreachable → refuse to start | NFR-I1: Vault dependency is a startup gate, not a runtime fallback. |
| Deferred | k8s service account (TokenReview API) | Clean post-MVP addition behind an `Authenticator` interface — no structural change required. |

## Error Contract

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Error path | Single path through `internal/errors/mapper.go` | Handlers never construct `ErrorEnvelope` inline. No silent failures possible by construction. |
| Adapter errors | Wrapped at adapter boundary (`fmt.Errorf("ollama: %w", err)`) | Provider internals never reach HTTP callers. |
| Panic recovery | Router-level middleware | Unhandled panics return 500 with a generic message — no stack traces in responses. |

## Infrastructure & Deployment

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Container base | `gcr.io/distroless/static` | CA certificates available for Vault TLS; no shell attack surface. |
| Helm chart location | Co-located in gateway repo (`/deploy/helm/`) | Not in terminus-infra — gateway owns its own deployment descriptor. |
| Config | Env vars → `GatewayConfig` struct at startup | K8s-native (ConfigMap/Secret injection). No hot-reload for MVP — restart to apply changes. Missing required vars = descriptive startup failure. |

## Testing

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Unit tests | Table-driven, co-located (`*_test.go`) | Consistent pattern across all packages; prevents single-case happy/sad splits. |
| Integration tests | `*_integration_test.go`, gated by `INTEGRATION=true` | Excluded from default `go test ./...`; opt-in for local dev and CI integration runs. |
| Contract tests | `internal/adapter/contract/suite_test.go` | Runs against all registered adapters; validates `Provider` interface compliance. Stub adapter must pass before any real adapter begins. |

## Deferred Decisions

| Area | Status | Path Forward |
|------|--------|-------------|
| k8s service account auth | Deferred post-MVP | Add behind `Authenticator` interface — no structural change |
| Rate limiting | Out of scope | Belongs in `terminus-inference-provider-routing` (per PRD scope boundary) |
| Service mesh auth delegation | Future | Istio/Linkerd mTLS — clean upgrade once mesh is in place |
| Config hot-reload | Deferred | No operational need for MVP; restart is acceptable |
