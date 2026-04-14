# Story 2.2: Resolve selects the highest-priority eligible provider

Status: ready-for-dev

## Story

As a **gateway operator**,
I want `Resolve` to evaluate the provider priority chain for the requested workload class and return the first eligible provider,
So that inference requests are dispatched to the correct provider without any routing logic in HTTP handlers.

## Acceptance Criteria

1. **Given** a `RoutingRequest` with `Profile: "interactive"` and the `interactive` profile has priority chain `[openai, ollama]`  
   **When** `Resolve` is called and `openai` is eligible  
   **Then** it returns `(&RoutingResponse{Provider: "openai", Degraded: false}, nil)`

2. **Given** a `RoutingRequest` with `Profile: "batch"` and the `batch` profile has priority chain `[ollama, openai]`  
   **When** `Resolve` is called and `ollama` is eligible  
   **Then** it returns `(&RoutingResponse{Provider: "ollama", Degraded: false}, nil)`

3. **Given** a profile with a three-provider chain `[a, b, c]` and provider `a` is unavailable  
   **When** `Resolve` is called  
   **Then** it returns `(&RoutingResponse{Provider: "b", Degraded: false}, nil)` (skips `a`, selects `b`)

4. **Given** `Resolve` is called concurrently by 100 goroutines with the same profile  
   **When** `go test -race` is run  
   **Then** zero race conditions are detected

## Tasks / Subtasks

- [ ] Implement the unexported `router` struct in `router.go` (AC: 1–4)
  - [ ] Fields: `profiles map[string][]string` (profile name → ordered provider names), `budget *BudgetTracker` (stub for Epic 3)
  - [ ] `Resolve(ctx context.Context, req *RoutingRequest) (*RoutingResponse, error)` method on `*router`
- [ ] Chain evaluation logic (AC: 1–3):
  - [ ] Look up `req.Profile` in `profiles` map
  - [ ] Iterate providers in priority order
  - [ ] Skip any provider where `BudgetTracker.IsExhausted(name)` returns true (stub returning false until Epic 3)
  - [ ] Return first eligible provider as `&RoutingResponse{Provider: name, Degraded: false}`
- [ ] Ensure provider name + workload class are NOT logged in this story (logging added in Story 4.1)
- [ ] Verify concurrent access with 100-goroutine test case (AC: 4)
- [ ] Verify `go build ./internal/routing/...` and `go test -race ./internal/routing/...` pass

## Dev Notes

- A provider is "eligible" at this stage (before Epic 3): not budget-exhausted and present in the profile chain. Use a `BudgetTracker` stub (returns `IsExhausted = false` always) so the check exists from day one — Epic 3 will fill it in
- `Resolve` uses `sync.RWMutex` via `BudgetTracker` for budget reads — `Resolve` should call `RLock`/`RUnlock` consistently (or let `BudgetTracker.IsExhausted` manage the lock)
- Priority chain is determined at config load time (`LoadProfiles`) — `Resolve` just iterates the slice in order; no dynamic reordering
- New providers and profiles added via config only (FR22, FR23) — no code changes needed for new entries; this is guaranteed by the map-based runtime state
- Performance: map lookup + slice iteration + counter check = < 1ms p99 (NFR-P1); no I/O, no network, no channel operations in `Resolve`
- The `Router` interface's `Resolve` method should NOT know about HTTP — it returns `(*RoutingResponse, error)` and the handler decides HTTP codes (Story 5.2)

### Project Structure Notes

- File: `internal/routing/router.go`
- `router` struct is unexported — only `LoadProfiles` constructs it and returns it as `Router` interface
- At this stage `BudgetTracker` (in `budget.go`) may be a stub struct with no-op methods; Epic 3 fully implements it

### References

- [Source: docs/terminus/inference/provider-routing/prd.md#FR1–FR4, FR22, FR23] — Priority chain; workload class routing; config-only extensibility
- [Source: docs/terminus/inference/provider-routing/architecture.md#Additional Requirements] — `sync.RWMutex` discipline; NFR-P1 < 1ms
- [Source: docs/terminus/inference/provider-routing/epics.md#Story 2.2] — All AC scenarios
- [Source: docs/terminus/inference/provider-routing/epics.md#Story 2.5] — Table-driven Resolve tests (next story for test coverage)

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
