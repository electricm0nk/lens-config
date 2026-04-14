# Story 3.1: BudgetTracker — per-provider token limit and RWMutex

Status: ready-for-dev

## Story

As a **gateway developer**,
I want `BudgetTracker` in `budget.go` to hold per-provider token limits and running spend with safe concurrent access,
So that budget reads in `Resolve` and budget writes in `RecordUsage` never race.

## Acceptance Criteria

1. **Given** `LoadProfiles` processes a provider with `daily_budget_usd: 5.00` and `price_per_1k_tokens: 0.002`  
   **When** the `BudgetTracker` is initialised for that provider  
   **Then** the computed token limit is `5.00 / 0.002 * 1000 = 2,500,000` tokens  
   **And** the initial running spend is `0`

2. **Given** a provider with `daily_budget_usd: 0`  
   **When** the `BudgetTracker` is initialised  
   **Then** no token limit is set for that provider — budget tracking is disabled (unlimited)

3. **Given** `BudgetTracker` is accessed concurrently by `Resolve` (RLock) and `RecordUsage` (Lock)  
   **When** `go test -race` exercises 100 concurrent readers and 10 concurrent writers  
   **Then** zero race conditions are detected

## Tasks / Subtasks

- [ ] Implement `BudgetTracker` struct in `budget.go` (AC: 1–3)
  - [ ] Internal `providerBudget` struct: `tokenLimit int64`, `tokenSpend int64`, `exhausted bool`, `mu sync.RWMutex`
  - [ ] `BudgetTracker` struct with `providers map[string]*providerBudget`
- [ ] Implement `NewBudgetTracker(providers []providerRuntime) *BudgetTracker` (AC: 1, 2)
  - [ ] For each provider with `daily_budget_usd > 0`: compute token limit = `int64(dailyBudgetUSD / pricePerKTokens * 1000)` (AC: 1)
  - [ ] For `daily_budget_usd == 0`: set `tokenLimit = 0` to signal unlimited (AC: 2)
- [ ] Implement `IsExhausted(provider string) bool` with `RLock`/`RUnlock` (AC: 3)
  - [ ] Returns `false` for unlimited providers regardless of spend
- [ ] Implement `AddSpend(provider string, tokens int64)` with `Lock`/`Unlock` (AC: 3)
  - [ ] No-op for unlimited providers
  - [ ] Otherwise: add tokens to `tokenSpend`; if `tokenSpend >= tokenLimit`, set `exhausted = true`
- [ ] Replace the `BudgetTracker` stub from Epic 2 with this real implementation (AC: 1)
- [ ] Verify all `router.go` calls to `IsExhausted` still compile (AC: 3)

## Dev Notes

- Token limit formula: `tokenLimit = int64(dailyBudgetUSD / pricePerKTokens * 1000)` — integer tokens computed once at startup; strictly an int64 counter from here on
- `daily_budget_usd: 0` = unlimited (FR12); `tokenLimit = 0` is the sentinel for "no limit"; `IsExhausted` must check `tokenLimit == 0` and return false immediately
- **Per-provider mutex discipline (NFR-P1, NFR-P2):**
  - `Resolve` path → calls `IsExhausted` → `RLock`/`RUnlock` on that provider's mutex
  - `RecordUsage` path → calls `AddSpend` → `Lock`/`Unlock` on that provider's mutex
  - One mutex PER provider, not one global mutex — allows concurrent reads for different providers
- Budget resets on process restart only (FR13) — no clock-based reset, no persistence; nothing to implement here
- `RecordUsage` (Story 3.2) will delegate to `AddSpend` — keep the interface compatible

### Project Structure Notes

- File: `internal/routing/budget.go`
- `BudgetTracker` and `providerBudget` are unexported to callers outside `internal/routing/` — only exposed via the `Router` interface methods
- `NewBudgetTracker` is called from `LoadProfiles` (config.go) after API key resolution

### References

- [Source: docs/terminus/inference/provider-routing/prd.md#FR8, FR9, FR12, FR13] — Dollar budget; token conversion; unlimited; restart reset
- [Source: docs/terminus/inference/provider-routing/architecture.md#Decision 1] — `sync.RWMutex` per provider; RLock/Lock discipline
- [Source: docs/terminus/inference/provider-routing/architecture.md#Additional Requirements] — Token formula; zero-value semantics; budget reset
- [Source: docs/terminus/inference/provider-routing/epics.md#Story 3.1] — All AC scenarios

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
