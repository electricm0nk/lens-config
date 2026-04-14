# Story 3.3: Resolve skips exhausted providers and surfaces degradation

Status: ready-for-dev

## Story

As an **operator**,
I want `Resolve` to automatically skip budget-exhausted providers and fall back to the next eligible provider with degradation metadata in the response,
So that interactive workloads degrade gracefully rather than failing when the primary provider is out of budget.

## Acceptance Criteria

1. **Given** the `interactive` profile has chain `[openai, ollama]` and `openai` is budget-exhausted  
   **When** `Resolve` is called with `Profile: "interactive"`  
   **Then** it returns `(&RoutingResponse{Provider: "ollama", Degraded: true, FallbackProvider: "ollama", DegradationReason: "budget_exhausted"}, nil)`

2. **Given** all providers in the chain are budget-exhausted  
   **When** `Resolve` is called  
   **Then** it returns `(nil, err)` where `errors.Is(err, ErrNoEligibleProvider)` is true  
   **And** no provider is silently selected beyond the declared chain

3. **Given** the primary provider is budget-exhausted and `Resolve` selects a fallback  
   **Then** the `Degraded` field on the response is `true` and `FallbackProvider` names the actual provider used

## Tasks / Subtasks

- [ ] Extend `Resolve` chain evaluation to use real `BudgetTracker.IsExhausted` (AC: 1–3)
  - [ ] Already calls `IsExhausted` from Story 2.2 stub — now the real implementation returns accurate state
- [ ] Add degradation metadata to `RoutingResponse` when fallback occurs (AC: 1, 3)
  - [ ] Track `firstEligible` = first non-exhausted provider encountered
  - [ ] If `firstEligible` != first-in-chain: set `Degraded: true`, `FallbackProvider: firstEligible`, `DegradationReason: DegradationReasonBudgetExhausted`
- [ ] All-exhausted path: return `ErrNoEligibleProvider` after iterating all providers (AC: 2)
  - [ ] Already implemented in Story 2.3 — verify it still fires when all are `IsExhausted == true`
- [ ] Write or extend `TestResolve` cases for exhaustion + fallback (AC: 1, 3)
  - [ ] Pre-exhaust openai via `BudgetTracker.AddSpend`, verify fallback to ollama with degradation fields

## Dev Notes

- Degradation logic: track which is the FIRST provider in the chain (index 0). If the selected provider is NOT the first one (index > 0 or `firstProvider != selectedProvider`), the response is degraded
- `DegradationReasonBudgetExhausted` = `"budget_exhausted"` — defined in `types.go` (Story 2.1); use the constant, not the string literal
- `FallbackProvider` should be the same as `Provider` in the response (the actual selected provider name) when degraded
- This story completes the integration of `BudgetTracker` with `Resolve` — after this story the routing pipeline is fully functional for Epic 3's scope
- Logging of degradation events is added in Story 4.1 — do NOT add log calls in this story (keep Stories isolated)

### Project Structure Notes

- File: `internal/routing/router.go` (extending `Resolve` chain evaluation)
- No new files required — this extends existing `Resolve` logic

### References

- [Source: docs/terminus/inference/provider-routing/prd.md#FR5, FR6] — Fallback on exhaustion; degradation fields in response
- [Source: docs/terminus/inference/provider-routing/architecture.md#Additional Requirements] — `DegradationReasonBudgetExhausted` constant
- [Source: docs/terminus/inference/provider-routing/epics.md#Story 3.3] — All AC scenarios

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
