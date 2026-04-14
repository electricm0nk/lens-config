# Story 3.2: RecordUsage accumulates actual token spend

**Initiative:** terminus-inference-provider-routing
**Epic:** 3 — Batch workloads can't exhaust cloud credits
**Status:** review

## Story

As a **gateway developer**,
I want `RecordUsage` to accumulate actual prompt+completion token counts from provider responses and mark providers exhausted at their limit,
So that dollar budget enforcement is based on real spend, not estimates.

## Acceptance Criteria

**Given** a provider with a token limit of 1000 and current spend of 0
**When** `RecordUsage(ctx, "openai", 400, 300)` is called (700 tokens)
**Then** the provider's running spend increases to 700 and `BudgetExhausted` remains `false`

**Given** the same provider now at 700 tokens of spend
**When** `RecordUsage(ctx, "openai", 200, 150)` is called (350 tokens, total 1050)
**Then** the provider's running spend is 1050, exceeding the 1000 limit
**And** `BudgetExhausted` is set to `true` for that provider

**Given** a provider with `daily_budget_usd: 0` (unlimited)
**When** `RecordUsage` is called with any token counts
**Then** no spend is tracked and `BudgetExhausted` remains `false` always

**Given** a provider response where `usage` is nil (token count unavailable), so `promptTokens: 0, completionTokens: 0`
**When** `RecordUsage` is called and the request body length is known
**Then** the estimated token count `len(requestBodyJSON) / 4` is used
**And** an `slog.Warn` is emitted indicating the estimate was used (no request body content in the log)

**Given** `RecordUsage` is called from the response path
**Then** it completes without any I/O or blocking operations — in-process counter update only

## Technical Notes

- Target repo: `terminus-inference-gateway`
- Implement `RecordUsage` on the router struct in `router.go` (or `budget.go`)
- `RecordUsage` signature: `RecordUsage(ctx context.Context, provider string, promptTokens, completionTokens int)`
- Nil-usage fallback: when both `promptTokens` and `completionTokens` are 0, apply `/ 4` estimate
  - The caller passes request body separately or keeper stores it — coordinate with story 5.2
- `slog.Warn` for nil-usage: log `"routing: RecordUsage: token counts unavailable, using estimate"` with `provider` and `estimated_tokens` — no body content
- Exhaustion check: after adding spend, compare `tokenSpent >= tokenLimit` and set `exhausted = true`

## Dependencies

- Story 3.1 (BudgetTracker struct)
