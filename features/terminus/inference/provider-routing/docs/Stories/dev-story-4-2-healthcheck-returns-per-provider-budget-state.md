# Story 4.2: HealthCheck returns per-provider budget state

**Initiative:** terminus-inference-provider-routing
**Epic:** 4 — Operator can observe routing health
**Status:** review

## Story

As an **operator**,
I want `HealthCheck` to return a `HealthStatus` with per-provider budget metrics,
So that the gateway's `/readyz` endpoint can report whether the routing layer is healthy.

## Acceptance Criteria

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

## Technical Notes

- Target repo: `terminus-inference-gateway`
- Implement `HealthCheck` on the router struct in `router.go`
- `HealthCheck` signature: `HealthCheck(ctx context.Context) *HealthStatus`
- USD conversion for `BudgetUsedUSD`: `float64(tokenSpent) / 1000 * pricePerKTokens`
- `AllExhausted` — `true` only when every tracked provider has `exhausted = true`
  - Unlimited providers (`tokenLimit = 0`) are never exhausted → contribute `false` to AllExhausted
- `HealthStatus.Degraded` — `true` if any provider is exhausted (partial degradation)
- Use `RLock` per provider entry — do not hold multiple locks simultaneously

## Dependencies

- Story 3.1 (BudgetTracker — has token spend values)
- Story 2.1 (HealthStatus / ProviderStatus types)
