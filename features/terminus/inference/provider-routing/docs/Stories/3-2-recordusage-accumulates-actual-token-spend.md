# Story 3.2: RecordUsage accumulates actual token spend

Status: ready-for-dev

## Story

As a **gateway developer**,
I want `RecordUsage` to accumulate actual prompt+completion token counts from provider responses and mark providers exhausted at their limit,
So that dollar budget enforcement is based on real spend, not estimates.

## Acceptance Criteria

1. **Given** a provider with a token limit of 1000 and current spend of 0  
   **When** `RecordUsage(ctx, "openai", 400, 300)` is called (700 tokens)  
   **Then** the provider's running spend increases to 700 and `BudgetExhausted` remains `false`

2. **Given** the same provider now at 700 tokens of spend  
   **When** `RecordUsage(ctx, "openai", 200, 150)` is called (350 tokens, total 1050)  
   **Then** the provider's running spend is 1050, exceeding the 1000 limit  
   **And** `BudgetExhausted` is set to `true` for that provider

3. **Given** a provider with `daily_budget_usd: 0` (unlimited)  
   **When** `RecordUsage` is called with any token counts  
   **Then** no spend is tracked and `BudgetExhausted` remains `false` always

4. **Given** a provider response where `usage` is nil (token count unavailable)  
   **When** `RecordUsage` is called with `promptTokens: 0, completionTokens: 0` and the request body length is known  
   **Then** the estimated token count `len(requestBodyJSON) / 4` is used  
   **And** an `slog.Warn` is emitted indicating the estimate was used (no request body content in the log)

5. **Given** `RecordUsage` is called from the response path  
   **Then** it completes without any I/O or blocking operations — in-process counter update only

## Tasks / Subtasks

- [ ] Implement `RecordUsage` on the `*router` struct in `router.go` (AC: 1–5)
  - [ ] `func (r *router) RecordUsage(ctx context.Context, provider string, promptTokens, completionTokens int)`
  - [ ] Total tokens = `promptTokens + completionTokens` (AC: 1, 2)
  - [ ] If total == 0 AND request body context value is available: use `len(bodyJSON)/4` as estimate, emit `slog.Warn` (AC: 4)
  - [ ] Delegate to `r.budget.AddSpend(provider, int64(total))` (AC: 1, 2, 3)
- [ ] Extend `BudgetTracker.AddSpend` to set `exhausted = true` when `tokenSpend >= tokenLimit` (AC: 2)
- [ ] Unlimited provider no-op in `AddSpend` (AC: 3)
- [ ] Nil-usage fallback with `slog.Warn` (AC: 4):
  - [ ] `slog.WarnContext(ctx, "routing: RecordUsage: token count unavailable, using estimate", "provider", provider, "estimated_tokens", estimated)`
  - [ ] The log line MUST NOT include request body content or any credential values (NFR-S1, NFR-S2)
- [ ] Verify `RecordUsage` is non-blocking — no I/O, no channels, no network calls (AC: 5)

## Dev Notes

- `RecordUsage` signature: `RecordUsage(ctx context.Context, provider string, promptTokens, completionTokens int)` — no return value (fire-and-forget update)
- The nil-usage fallback (`len(requestBodyJSON) / 4`) requires passing the raw request body length somehow; the caller (Story 5.2) is responsible for providing this; one approach is a context key `bodyLenKey` that the handler sets before calling `RecordUsage`; design the context key in this story and document it for Story 5.2
- `slog.Warn` on nil-usage: log ONLY `provider` name and `estimated_tokens` count — never the body content (NFR-S2 prohibits logging request content)
- Non-blocking guarantee (NFR-P2): `RecordUsage` must complete with only in-memory operations — no goroutine launch, no channel send, no mutex wait beyond sub-microsecond RWMutex contention
- `AddSpend` atomicity: increment then check threshold in the same `Lock` scope — no TOCTOU window

### Project Structure Notes

- Files: `internal/routing/router.go` (RecordUsage method), `internal/routing/budget.go` (AddSpend exhaustion logic)
- Context key for body length: define as an unexported type in `router.go` or `config.go` — e.g., `type contextKey int; const bodyLenContextKey contextKey = iota`

### References

- [Source: docs/terminus/inference/provider-routing/prd.md#FR10, FR11, FR12, NFR-P2] — RecordUsage; cumulative tracking; unlimited; non-blocking
- [Source: docs/terminus/inference/provider-routing/architecture.md#Additional Requirements] — "Token fallback when usage nil: `len(requestBodyJSON) / 4`; log `slog.Warn`"
- [Source: docs/terminus/inference/provider-routing/epics.md#Story 3.2] — All AC scenarios

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
