---
phase: devproposal
initiative: terminus-inference-provider-routing
generatedFrom: epics.md
date: 2026-04-07
---

# terminus-inference-provider-routing — Story Index

All 21 stories are fully defined in [epics.md](epics.md) with complete Given/When/Then acceptance criteria. This index provides the stakeholder-readable summary used for sprint planning.

---

## Epic 1: Operator can configure routing policy in YAML

| Story | Title | FRs | Key Deliverable |
|---|---|---|---|
| 1.1 | Package scaffold and sentinel errors | – | `internal/routing/` package with 5 source files; `go build` passes; sentinel errors exported |
| 1.2 | LoadProfiles parses and validates the YAML config | FR14, FR16, FR17 | Hard startup errors with field-level messages; schema version enforcement; slog.Info on success |
| 1.3 | LoadProfiles resolves API keys from environment variables | FR15, NFR-S3 | API keys resolved from env at startup; never in logs, exports, or error messages |
| 1.4 | Config tests with testdata fixtures | NFR-I1 | `internal/routing/testdata/` per-scenario YAMLs; `go test -race` passes |

---

## Epic 2: Gateway can route requests to the correct provider

| Story | Title | FRs | Key Deliverable |
|---|---|---|---|
| 2.1 | RoutingRequest, RoutingResponse, and shared types | – | `types.go` with all shared types + degradation constants |
| 2.2 | Resolve selects the highest-priority eligible provider | FR1, FR2, FR3, FR4, FR22, FR23 | Priority chain evaluation; first-eligible selection; race-safe |
| 2.3 | Resolve returns clean sentinel errors | FR7, NFR-R1, NFR-R2 | ErrInvalidWorkloadClass, ErrNoEligibleProvider; panic recovery via defer/recover |
| 2.4 | HTTP error mapping for routing sentinel errors | – | ErrNoEligibleProvider → 503; ErrInvalidWorkloadClass → 400 in `internal/errors` |
| 2.5 | router_test.go — table-driven Resolve tests | NFR-P1 | All Resolve paths; `-race` clean |

---

## Epic 3: Batch workloads can't exhaust cloud credits

| Story | Title | FRs | Key Deliverable |
|---|---|---|---|
| 3.1 | BudgetTracker — per-provider token limit and RWMutex | FR8, FR9, FR12, FR13 | `sync.RWMutex`; dollar→token conversion; unlimited semantics |
| 3.2 | RecordUsage accumulates actual token spend | FR10, FR11, NFR-P2 | Cumulative tracking; exhaustion threshold; nil-usage fallback estimate + slog.Warn |
| 3.3 | Resolve skips exhausted providers and surfaces degradation | FR5, FR6 | Fallback selection; RoutingResponse.Degraded, FallbackProvider, DegradationReason |
| 3.4 | budget_test.go — table-driven budget and fallback tests | NFR-R3 | Threshold, zero-budget, concurrent writes; `-race` clean |

---

## Epic 4: Operator can observe routing health

| Story | Title | FRs | Key Deliverable |
|---|---|---|---|
| 4.1 | Resolve emits structured log per request | FR18, FR19, NFR-S2 | slog.Debug per Resolve call; no PII; degradation in log |
| 4.2 | HealthCheck returns per-provider budget state | FR20, NFR-P3, NFR-S1 | HealthStatus + ProviderStatus; < 5ms; no credentials in output |
| 4.3 | HealthCheck and logging tests | NFR-S1, NFR-S2 | Healthy/degraded/exhausted coverage; PII-safety verified by log capture |

---

## Epic 5: Gateway can deploy with routing enabled

| Story | Title | FRs | Key Deliverable |
|---|---|---|---|
| 5.1 | main.go wires LoadProfiles at startup | FR1 | LoadProfiles called before server starts; hard exit on error; no global Router variable |
| 5.2 | HTTP handler calls Resolve and RecordUsage | FR1 | Per-request routing; RecordUsage on every response; degradation in response body |
| 5.3 | /readyz wires HealthCheck | FR21 | AllExhausted: true → HTTP 503 from /readyz |
| 5.4 | Deployment artifacts — config and Helm/ArgoCD | FR22, FR23 | ROUTING_CONFIG_PATH in Helm; sample routing-config.yaml with 2 providers + 2 profiles |
| 5.5 | Smoke test — routing layer active in deployed gateway | – | Interactive → OpenAI; batch → Ollama; budget exhaustion → fallback + degraded |

---

## Summary

| Metric | Count |
|---|---|
| Epics | 5 |
| Stories | 21 |
| FRs covered | 23/23 |
| NFRs covered | 10/10 |
| Architecture constraints | 15/15 |
| Race-detector gate | `go test -race ./internal/routing/...` |
| Implementation language | Go — `internal/routing/` inside `terminus-inference-gateway` |
