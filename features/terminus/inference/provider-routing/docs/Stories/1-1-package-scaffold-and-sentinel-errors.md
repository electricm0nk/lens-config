# Story 1.1: Package scaffold and sentinel errors

Status: ready-for-dev

## Story

As a **gateway developer**,
I want the `internal/routing/` package to exist with the full file scaffold and exported sentinel errors,
So that all subsequent routing stories have a valid Go package to build into.

## Acceptance Criteria

1. **Given** the gateway module exists  
   **When** `internal/routing/` is created with `router.go`, `budget.go`, `config.go`, `errors.go`, `types.go`  
   **Then** `go build ./internal/routing/...` succeeds with no errors

2. **And** `errors.go` exports `ErrNoEligibleProvider`, `ErrBudgetExhausted`, `ErrInvalidWorkloadClass`

3. **And** `config.go` exports `const SupportedSchemaVersion = 1`

4. **And** `router.go` declares the `Router` interface with `Resolve`, `RecordUsage`, `HealthCheck` signatures matching the architecture specification

5. **And** the package compiles on the same Go version as the gateway with zero new external dependencies (only `gopkg.in/yaml.v3`)

## Tasks / Subtasks

- [ ] Create `internal/routing/` directory (AC: 1)
- [ ] Create `errors.go` with three sentinel errors (AC: 2)
  - [ ] `var ErrNoEligibleProvider = errors.New("routing: no eligible provider")`
  - [ ] `var ErrBudgetExhausted = errors.New("routing: budget exhausted")`
  - [ ] `var ErrInvalidWorkloadClass = errors.New("routing: invalid workload class")`
- [ ] Create `config.go` with `SupportedSchemaVersion = 1` constant (AC: 3)
- [ ] Create `router.go` declaring the `Router` interface (AC: 4)
  - [ ] `Resolve(ctx context.Context, req *RoutingRequest) (*RoutingResponse, error)`
  - [ ] `RecordUsage(ctx context.Context, provider string, promptTokens, completionTokens int)`
  - [ ] `HealthCheck(ctx context.Context) *HealthStatus`
- [ ] Create `budget.go` as empty package stub (AC: 1)
- [ ] Create `types.go` as empty package stub (AC: 1)
- [ ] Verify `go build ./internal/routing/...` passes (AC: 1, 5)
- [ ] Verify no new external deps introduced beyond `gopkg.in/yaml.v3` (AC: 5)

## Dev Notes

- Package path: `internal/routing/` inside `terminus-inference-gateway` module — NOT a new module or repo
- All 5 files must exist at the end of this story: `router.go`, `budget.go`, `config.go`, `errors.go`, `types.go`
- `router.go` declares the `Router` interface as the sole integration boundary (NFR-I1); `types.go` will hold the concrete types (Story 2.1)
- The `Router` interface in `router.go` must reference types that will be defined in `types.go` — use forward-compatible stubs or define placeholder structs; prefer defining `RoutingRequest`, `RoutingResponse`, `HealthStatus` as empty structs here and flesh them out in Story 2.1
- Error wrapping convention for this package: `fmt.Errorf("routing: {action}: %w", err)` — applies from Story 1.2 onward; sentinel errors in `errors.go` are base values only
- `gopkg.in/yaml.v3` — verify it is already in `go.mod`; if not, run `go get gopkg.in/yaml.v3` before adding any import

### Project Structure Notes

- Target repo: `TargetProjects/terminus/inference` (the gateway)
- Package: `internal/routing/` — enforces encapsulation; only gateway HTTP handlers can import it
- No exported packages from inside `internal/routing/` beyond what `router.go` exposes via the `Router` interface
- Must compile cleanly against the same Go version as the gateway — check `go.mod` for `go` directive

### References

- [Source: docs/terminus/inference/provider-routing/architecture.md#Project Context] — Package path, file list, no-new-module decision
- [Source: docs/terminus/inference/provider-routing/architecture.md#Core Architectural Decisions] — Router interface as sole boundary, stdlib-only rationale
- [Source: docs/terminus/inference/provider-routing/prd.md#FR14, FR16] — Config at startup; `SupportedSchemaVersion`
- [Source: docs/terminus/inference/provider-routing/epics.md#Epic 1] — Scaffold scope: all 5 source files + test files

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
