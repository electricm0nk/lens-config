# Story 3.1: BudgetTracker — per-provider token limit and RWMutex

**Initiative:** terminus-inference-provider-routing
**Epic:** 3 — Batch workloads can't exhaust cloud credits
**Status:** review

## Story

As a **gateway developer**,
I want `BudgetTracker` in `budget.go` to hold per-provider token limits and running spend with safe concurrent access,
So that budget reads in `Resolve` and budget writes in `RecordUsage` never race.

## Acceptance Criteria

**Given** `LoadProfiles` processes a provider with `daily_budget_usd: 5.00` and `price_per_1k_tokens: 0.002`
**When** the `BudgetTracker` is initialised for that provider
**Then** the computed token limit is `5.00 / 0.002 * 1000 = 2,500,000` tokens
**And** the initial running spend is `0`

**Given** a provider with `daily_budget_usd: 0`
**When** the `BudgetTracker` is initialised
**Then** no token limit is set for that provider — budget tracking is disabled (unlimited)

**Given** `BudgetTracker` is accessed concurrently by `Resolve` (RLock) and `RecordUsage` (Lock)
**When** `go test -race` exercises 100 concurrent readers and 10 concurrent writers
**Then** zero race conditions are detected

## Technical Notes

- Target repo: `terminus-inference-gateway`
- File: `internal/routing/budget.go`
- `BudgetTracker` struct (unexported or exported as needed):
  ```go
  type budgetEntry struct {
      mu          sync.RWMutex
      tokenLimit  int64   // 0 = unlimited
      tokenSpent  int64
      exhausted   bool
  }
  ```
- Token limit formula: `int64(dailyBudgetUSD / pricePerKTokens * 1000)`
- `daily_budget_usd: 0` → `tokenLimit = 0` → budget tracking disabled
- `Resolve` reads exhaustion state under `RLock`
- `RecordUsage` updates spend under `Lock`
- One mutex per provider — not a single global lock

## Dependencies

- Story 1.2 (LoadProfiles — budget values parsed from config)
