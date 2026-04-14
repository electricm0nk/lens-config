# Story 1.1: Package scaffold and sentinel errors

**Initiative:** terminus-inference-provider-routing
**Epic:** 1 — Operator can configure routing policy in YAML
**Status:** review

## Story

As a **gateway developer**,
I want the `internal/routing/` package to exist with the full file scaffold and exported sentinel errors,
So that all subsequent routing stories have a valid Go package to build into.

## Acceptance Criteria

**Given** the gateway module exists
**When** `internal/routing/` is created with `router.go`, `budget.go`, `config.go`, `errors.go`, `types.go`
**Then** `go build ./internal/routing/...` succeeds with no errors
**And** `errors.go` exports `ErrNoEligibleProvider`, `ErrBudgetExhausted`, `ErrInvalidWorkloadClass`
**And** `config.go` exports `const SupportedSchemaVersion = 1`
**And** `router.go` declares the `Router` interface with `Resolve`, `RecordUsage`, `HealthCheck` signatures matching the architecture specification
**And** the package compiles on the same Go version as the gateway with zero new external dependencies (only `gopkg.in/yaml.v3`)

## Technical Notes

- Target repo: `terminus-inference-gateway`
- Package path: `internal/routing/`
- Files to create: `router.go`, `budget.go`, `config.go`, `errors.go`, `types.go`
- No new module — package lives inside existing gateway module
- `Router` interface signatures (from architecture):
  - `Resolve(ctx context.Context, req *RoutingRequest) (*RoutingResponse, error)`
  - `RecordUsage(ctx context.Context, provider string, promptTokens, completionTokens int)`
  - `HealthCheck(ctx context.Context) *HealthStatus`
- Sentinel errors use stdlib `errors.New`; wrapping uses `fmt.Errorf("routing: …: %w", err)` convention
- `SupportedSchemaVersion = 1` — required in all YAML configs

## Dependencies

- None — this is the first story; all others depend on it
