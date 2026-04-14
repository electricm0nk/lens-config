# Story 2.2: Resolve selects the highest-priority eligible provider

**Initiative:** terminus-inference-provider-routing
**Epic:** 2 — Gateway can route requests to the correct provider
**Status:** review

## Story

As a **gateway operator**,
I want `Resolve` to evaluate the provider priority chain for the requested workload class and return the first eligible provider,
So that inference requests are dispatched to the correct provider without any routing logic in HTTP handlers.

## Acceptance Criteria

**Given** a `RoutingRequest` with `Profile: "interactive"` and the `interactive` profile has priority chain `[openai, ollama]`
**When** `Resolve` is called and `openai` is eligible
**Then** it returns `(&RoutingResponse{Provider: "openai", Degraded: false}, nil)`

**Given** a `RoutingRequest` with `Profile: "batch"` and the `batch` profile has priority chain `[ollama, openai]`
**When** `Resolve` is called and `ollama` is eligible
**Then** it returns `(&RoutingResponse{Provider: "ollama", Degraded: false}, nil)`

**Given** a profile with a three-provider chain `[a, b, c]` and provider `a` is unavailable
**When** `Resolve` is called
**Then** it returns `(&RoutingResponse{Provider: "b", Degraded: false}, nil)` (skips `a`, selects `b`)

**Given** `Resolve` is called concurrently by 100 goroutines with the same profile
**When** `go test -race` is run
**Then** zero race conditions are detected

## Technical Notes

- Target repo: `terminus-inference-gateway`
- Implement `Resolve` on the concrete router struct in `router.go`
- Priority chain evaluation: iterate providers in declared order, skip budget-exhausted ones
- `eligible` check in MVP: provider is not budget-exhausted (no liveness/ping check)
- Profile lookup: `map[string][]string` (profile name → ordered provider names) built at `LoadProfiles`
- Thread safety: use `RLock` on `BudgetTracker` during `Resolve`; no Lock needed for read-only chain iteration
- `Degraded: false` on the happy path — no fallback occurred

## Dependencies

- Story 1.2 (LoadProfiles / config)
- Story 2.1 (types)
- Story 3.1 (BudgetTracker, needed to check exhaustion) — can stub with always-eligible initially
