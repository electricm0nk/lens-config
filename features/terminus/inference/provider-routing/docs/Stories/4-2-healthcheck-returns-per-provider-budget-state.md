# Story 4.2: HealthCheck returns per-provider budget state

Status: ready-for-dev

## Story

As an **operator**,
I want `HealthCheck` to return a `HealthStatus` with per-provider budget metrics,
So that the gateway's `/readyz` endpoint can report whether the routing layer is healthy.

## Acceptance Criteria

1. **Given** two providers loaded — `openai` (budget limit 5.00 USD, 2.30 USD used) and `ollama` (unlimited)  
   **When** `HealthCheck(ctx)` is called  
   **Then** it returns a `HealthStatus` with `Providers` containing one entry per provider  
   **And** the `openai` `ProviderStatus` has `BudgetLimitUSD: 5.00`, `BudgetUsedUSD: 2.30`, `BudgetExhausted: false`  
   **And** the `ollama` `ProviderStatus` has `BudgetLimitUSD: 0`, `BudgetUsedUSD: 0`, `BudgetExhausted: false`  
   **And** `HealthStatus.AllExhausted` is `false`

2. **Given** all providers are budget-exhausted  
   **When** `HealthCheck(ctx)` is called  
   **Then** `HealthStatus.AllExhausted` is `true` and `HealthStatus.Degraded` is `true`

3. **Given** `HealthCheck` is called from the gateway's `/readyz` handler  
   **Then** it completes in under 5ms — reads in-memory state only, no network calls

4. **Given** `HealthCheck` returns a `HealthStatus`  
   **Then** no API key value or credential appears in any field of `HealthStatus` or any `ProviderStatus`

## Tasks / Subtasks

- [ ] Implement `HealthCheck` on `*router` in `router.go` (AC: 1–4)
  - [ ] `func (r *router) HealthCheck(ctx context.Context) *HealthStatus`
  - [ ] Iterate `r.budget.providers`; for each: build `ProviderStatus` from current `BudgetTracker` state (AC: 1)
  - [ ] `BudgetLimitUSD`: convert `tokenLimit` back to USD (`float64(tokenLimit) * pricePerKTokens / 1000`) (AC: 1)
  - [ ] `BudgetUsedUSD`: convert `tokenSpend` back to USD similarly (AC: 1)
  - [ ] `BudgetExhausted`: read `exhausted` field (AC: 1, 2) — use `RLock` to read
  - [ ] `AllExhausted`: `true` if ALL providers have `exhausted == true` (AC: 2)
  - [ ] `HealthStatus.Degraded`: `true` if `AllExhausted == true` (AC: 2)
- [ ] Expose `pricePerKTokens` per provider in `BudgetTracker` (needed for USD back-conversion) (AC: 1)
- [ ] Verify no credential values are present in any field of `HealthStatus` or `ProviderStatus` (AC: 4)
- [ ] Verify `HealthCheck` uses only in-memory reads (AC: 3)

## Dev Notes

- `HealthCheck` reads use `RLock` (same as `IsExhausted`) — non-exclusive, completes in microseconds (NFR-P3: < 5ms)
- `BudgetLimitUSD` for unlimited providers (`tokenLimit == 0`): return `0` (consistent with config semantics; unlimited = zero declared budget)
- `BudgetUsedUSD` for unlimited providers: return `0` (no tracking performed)
- Security invariant (NFR-S1): `ProviderStatus` contains ONLY budget metrics: `Name`, `BudgetLimitUSD`, `BudgetUsedUSD`, `BudgetExhausted` — never `APIKey`, `resolvedAPIKey`, or any credential field
- `HealthStatus.Degraded` field semantics: `true` when the routing layer is in a degraded state (all exhausted); this is distinct from per-request degradation in `RoutingResponse`
- `AllExhausted` is used by Story 5.3 to trigger `/readyz` → 503 — keep this field accurate

### Project Structure Notes

- File: `internal/routing/router.go` (implementing `HealthCheck` on `*router`)
- `BudgetTracker` may need a `ProviderSnapshot(name string) (limit, spend int64, pricePerK float64, exhausted bool)` method to expose state cleanly for `HealthCheck`

### References

- [Source: docs/terminus/inference/provider-routing/prd.md#FR20, NFR-P3, NFR-S1] — HealthCheck; < 5ms; no keys in output
- [Source: docs/terminus/inference/provider-routing/architecture.md#Additional Requirements] — `HealthStatus` + `ProviderStatus` struct shapes
- [Source: docs/terminus/inference/provider-routing/epics.md#Story 4.2] — All AC scenarios

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
