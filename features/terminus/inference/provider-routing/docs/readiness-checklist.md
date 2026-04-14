# Implementation Readiness Checklist — terminus-inference-provider-routing (DevProposal)

**Date:** 2026-07-17
**Initiative:** terminus-inference-provider-routing
**Audience:** medium (lead review)
**Phase:** devproposal
**Mode:** validate (run against committed artifacts)
**Reviewer:** Lord Commander Creed (PM)

---

## Document Inventory

| Artifact | Path | Status |
|---|---|---|
| PRD | `docs/terminus/inference/provider-routing/prd.md` | ✅ Present — 23 FRs + 10 NFRs |
| Architecture | `docs/terminus/inference/provider-routing/architecture.md` | ✅ Present — all gaps resolved, Go package layout, concurrency model |
| Adversarial Review | `docs/terminus/inference/provider-routing/adversarial-review-report.md` | ✅ PASS_WITH_NOTES |
| Epics & Stories | `docs/terminus/inference/provider-routing/epics.md` | ✅ Present — 5 epics, 21 stories |

---

## FR Coverage Validation

All 23 functional requirements from the PRD are mapped to epics and stories.

| FR | Story Coverage | Description |
|---|---|---|
| FR1 | 2.2, 5.1, 5.2 | Gateway resolves provider via Router without routing logic in handlers |
| FR2 | 2.2 | Priority chain evaluation per workload class |
| FR3 | 2.2, 5.4 | `interactive` and `batch` workload classes supported |
| FR4 | 2.2 | Each workload class has independently configurable chain |
| FR5 | 3.3 | Fallback to next eligible provider when first-priority exhausted |
| FR6 | 3.3 | `degraded`, `fallback_provider`, `degradation_reason` in RoutingResponse |
| FR7 | 2.3 | `ErrNoEligibleProvider` returned when no eligible provider; no silent routing |
| FR8 | 3.1 | `daily_budget_usd` per provider in YAML config |
| FR9 | 3.1 | Dollar→token conversion at LoadProfiles startup |
| FR10 | 3.2 | `RecordUsage` called with actual prompt+completion counts |
| FR11 | 3.2 | Cumulative per-provider tracking; exhaustion threshold enforcement |
| FR12 | 3.1 | `daily_budget_usd: 0` = unlimited; no tracking performed |
| FR13 | 3.1 | Budget resets on process restart — in-process only, no persistence |
| FR14 | 1.2 | YAML config with `schema_version` field; unsupported version = hard error |
| FR15 | 1.3 | API keys referenced by env var name; resolved at startup by `LoadProfiles` |
| FR16 | 1.2, 1.3 | Field-level validation; process exits with field-naming error on any invalid config |
| FR17 | 1.2 | `slog.Info` on successful config load confirming file path |
| FR18 | 4.1 | Provider, workload class, degradation status logged per `Resolve` call |
| FR19 | 4.1, 4.3 | No prompt/response/credential content ever logged — PII-safety verified by test |
| FR20 | 4.2 | `HealthCheck` returns `HealthStatus` with per-provider budget state |
| FR21 | 5.3 | `/readyz` calls `HealthCheck`; `AllExhausted: true` → HTTP 503 |
| FR22 | 2.2, 5.4 | New provider added via config only — no code change |
| FR23 | 2.2, 5.4 | New workload class added via config only — no code change |

**FR Coverage: 23/23 — COMPLETE** ✅

---

## NFR Coverage Validation

| NFR | Story Coverage | Verification Method |
|---|---|---|
| NFR-P1: `Resolve` < 1ms p99 | 2.2 | Map lookup + counter read; no I/O; asserted in story AC |
| NFR-P2: `RecordUsage` non-blocking | 3.2 | In-process counter update only; no I/O; asserted in story AC |
| NFR-P3: `HealthCheck` < 5ms | 4.2 | In-memory state reads only; asserted in story AC |
| NFR-S1: No keys in logs/HealthStatus | 1.3, 4.1, 4.2, 4.3 | Test verifies no credential values in any exported type or log record |
| NFR-S2: No prompt/response in logs | 4.1, 4.3 | slog allow-list enforced; PII-safety test in 4.3 |
| NFR-S3: Keys resolved once at startup | 1.3 | `LoadProfiles` resolves all keys; resolved values in unexported fields only |
| NFR-S4: No routing-layer auth | 2.2 | Caller identity not in `RoutingRequest`; out of scope for routing MVP |
| NFR-I1: Router interface boundary | 1.1 | `Router` interface is sole integration contract; no direct struct imports |
| NFR-I2: Same Go version as gateway | 1.1 | No new Go version requirement; stdlib + `yaml.v3` only |
| NFR-I3: No dep conflicts | 1.1 | One external dep: `gopkg.in/yaml.v3`; no gateway conflict |
| NFR-R1: Panic recovery in Resolve | 2.3 | Named-return `defer/recover` asserted; gateway never crashes on routing panic |
| NFR-R2: ErrInvalidWorkloadClass clean | 2.3 | Asserted: sentinel returned, no panic, includes unrecognized class name |
| NFR-R3: No silent routing on exhaustion | 2.3, 3.3 | All-exhausted returns `ErrNoEligibleProvider`; no provider chosen beyond chain |

**NFR Coverage: 10/10 — COMPLETE** ✅

---

## Architecture Compliance

| Constraint | Status | Story |
|---|---|---|
| Package in `internal/routing/` — no new module | ✅ | 1.1 |
| Files: `router.go`, `budget.go`, `config.go`, `errors.go`, `types.go` | ✅ | 1.1 |
| Stdlib only + `gopkg.in/yaml.v3` | ✅ | 1.1 |
| `sync.RWMutex` per provider; RLock(Resolve) / Lock(RecordUsage) | ✅ | 3.1, 3.2 |
| `ROUTING_CONFIG_PATH` is only injection point; empty = hard error | ✅ | 1.2, 5.1 |
| Three sentinel errors declared | ✅ | 1.1 |
| `ErrNoEligibleProvider` → HTTP 503 mapping | ✅ | 2.4 |
| `ErrInvalidWorkloadClass` → HTTP 400 mapping | ✅ | 2.4 |
| Error wrapping: `fmt.Errorf("routing: …: %w", err)` | ✅ | 1.2, 2.3 |
| Panic recovery: named-return defer/recover | ✅ | 2.3 |
| Token fallback: `len(requestBodyJSON)/4` + `slog.Warn` | ✅ | 3.2 |
| `SupportedSchemaVersion = 1` constant | ✅ | 1.1, 1.2 |
| Budget reset = process restart (no clock-based reset) | ✅ | 3.1 |
| `testdata/` fixtures for config tests; `-race` CI gate | ✅ | 1.4 |
| `HealthStatus.AllExhausted: true` → `/readyz` HTTP 503 | ✅ | 5.3 |

**Architecture Coverage: 15/15 — COMPLETE** ✅

---

## Story Quality Validation

### Story Independence and Forward-Dependency Check

Each story was verified to depend only on previous stories within its epic. No forward references found.

| Epic | Stories | Forward Deps | Status |
|---|---|---|---|
| Epic 1 | 1.1 → 1.2 → 1.3 → 1.4 | None | ✅ |
| Epic 2 | 2.1 → 2.2 → 2.3 → 2.4 → 2.5 | None | ✅ |
| Epic 3 | 3.1 → 3.2 → 3.3 → 3.4 | None | ✅ |
| Epic 4 | 4.1 → 4.2 → 4.3 | None | ✅ |
| Epic 5 | 5.1 → 5.2 → 5.3 → 5.4 → 5.5 | None | ✅ |

### Epic Independence Check

| Epic | Can stand alone after? | Status |
|---|---|---|
| Epic 1 (config) | Foundation for all — standalone compile target | ✅ |
| Epic 2 (routing) | Fully functional `Resolve` with no budget tracking | ✅ |
| Epic 3 (budget) | Budget layer adds to Epic 2 without modifying its contract | ✅ |
| Epic 4 (observability) | Adds logging/health to existing types; no Epic 3 hard dep | ✅ |
| Epic 5 (deployment) | Integration epic; requires 1–4, but they deliver full `Router` | ✅ |

### Story Completability

Each story:
- ✅ Completable by a single dev agent without prerequisite coordination
- ✅ Has Given/When/Then acceptance criteria
- ✅ References specific FRs it implements
- ✅ Includes required technical details (type names, error names, log fields)
- ✅ Includes `-race` test requirement where applicable
- ✅ Specifies no credentials in fixtures or test expectations

---

## Cross-Cutting Concerns

| Concern | Coverage | Notes |
|---|---|---|
| Security — credential leakage | Stories 1.3, 4.1, 4.2, 4.3 | Resolved keys in unexported fields; PII-safe tests verify no credential value in any exported surface |
| Security — error message safety | Stories 1.2, 1.3, 2.3 | Errors name fields/providers/env var names only — no credential values in error text |
| Concurrency — race safety | Stories 2.5, 3.4, 4.3 | All test stories specify `-race`; BudgetTracker RWMutex discipline verified |
| Observability — log completeness | Stories 4.1, 4.3 | Every Resolve path (happy, degraded, error) emits a structured log line |
| Resilience — panic containment | Story 2.3 | Named-return defer/recover; gateway process never crashes on routing panic |
| Testability — hermetic fixtures | Story 1.4 | `testdata/` files per scenario; no real API keys; env vars set inline in test |

---

## Readiness Verdict

| Dimension | Status | Notes |
|---|---|---|
| FR Coverage | ✅ 23/23 COMPLETE | All FRs mapped to stories with acceptance criteria |
| NFR Coverage | ✅ 10/10 COMPLETE | All NFRs addressed with verifiable acceptance criteria |
| Architecture Compliance | ✅ 15/15 COMPLETE | All architectural constraints reflected in stories |
| Story Quality | ✅ PASS | No forward deps, single-agent completable, clear ACs |
| Security | ✅ PASS | Credential safety coverage in three separate stories with test verification |
| Concurrency | ✅ PASS | RWMutex discipline + race-detector CI gate across all concurrent paths |
| Deployment | ✅ PASS | Epic 5 delivers complete go-live path with smoke test and deployment artifacts |

**Overall Readiness: ✅ READY FOR IMPLEMENTATION**

Stories are sequenced and self-contained. Sprint planning can begin with Epic 1 from story 1.1.
