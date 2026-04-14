# Story 3.3: Resolve skips exhausted providers and surfaces degradation

**Initiative:** terminus-inference-provider-routing
**Epic:** 3 — Batch workloads can't exhaust cloud credits
**Status:** review

## Story

As an **operator**,
I want `Resolve` to automatically skip budget-exhausted providers and fall back to the next eligible provider with degradation metadata in the response,
So that interactive workloads degrade gracefully rather than failing when the primary provider is out of budget.

## Acceptance Criteria

**Given** the `interactive` profile has chain `[openai, ollama]` and `openai` is budget-exhausted
**When** `Resolve` is called with `Profile: "interactive"`
**Then** it returns:
```go
&RoutingResponse{
    Provider:          "ollama",
    Degraded:          true,
    FallbackProvider:  "ollama",
    DegradationReason: DegradationReasonBudgetExhausted,
}
```

**Given** all providers in the chain are budget-exhausted
**When** `Resolve` is called
**Then** it returns `(nil, err)` where `errors.Is(err, ErrNoEligibleProvider)` is true
**And** no provider is silently selected beyond the declared chain

**Given** the primary provider is budget-exhausted and `Resolve` selects a fallback
**Then** the `Degraded` field on the response is `true` and `FallbackProvider` names the actual provider used

## Technical Notes

- Target repo: `terminus-inference-gateway`
- Modify `Resolve` in `router.go` — extend priority chain iteration to check `budgetEntry.exhausted`
- First non-exhausted provider = selected provider
- If the selected provider is NOT the first in the chain → set `Degraded: true`, `FallbackProvider: selectedProvider`, `DegradationReason: DegradationReasonBudgetExhausted`
- If no provider is non-exhausted → return `ErrNoEligibleProvider`
- `RLock` on each provider's budget entry during evaluation — do not hold lock across multiple providers simultaneously

## Dependencies

- Story 2.2 (Resolve priority chain base)
- Story 3.1 (BudgetTracker with exhausted flag)
- Story 3.2 (RecordUsage sets exhausted flag)
