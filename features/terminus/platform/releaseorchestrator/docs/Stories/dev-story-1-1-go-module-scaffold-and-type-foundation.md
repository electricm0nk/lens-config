# Story 1.1: Go Module Scaffold and Type Foundation

Status: ready-for-dev

## Story

As a platform developer,
I want a Go module initialised at `github.com/electricm0nk/terminus-platform` with `internal/types/` declaring `ModulePayload[S any]`, `RunContext`, `ReleaseInput`, and `ReleaseResult`,
so that all subsequent stories have a typed foundation to build on with no `interface{}` in signatures.

## Acceptance Criteria

1. `go.mod` exists at repo root declaring `module github.com/electricm0nk/terminus-platform` and `go 1.25`
2. `go.sum` is present (may be empty until dependencies added in story 1.2)
3. `internal/types/module.go` declares `RunContext`, `ModulePayload[S any]`, and `Module` interface matching the TypeScript originals semantically
4. `internal/types/release.go` declares `ReleaseInput{Service, SHA, Env string; ProvisionDB bool}` and `ReleaseResult{Status, CompletedAt string; SmokeTestPassed bool}`
5. `internal/types/module_test.go` and `internal/types/release_test.go` exist with 100% coverage on type construction and zero-value behaviour
6. `go test ./internal/types/...` passes
7. TypeScript files are NOT yet removed (TypeScript scaffold coexists; removal is Story 1.4)

## Tasks / Subtasks

- [ ] Run `go mod init github.com/electricm0nk/terminus-platform` at repo root (AC: 1)
- [ ] Create `internal/types/module.go` with `RunContext`, `ModulePayload[S any]`, `Module` interface (AC: 3)
  - [ ] Write `module_test.go` FIRST (TDD) covering zero-value and construction
- [ ] Create `internal/types/release.go` with `ReleaseInput` and `ReleaseResult` structs (AC: 4)
  - [ ] Write `release_test.go` FIRST (TDD) covering zero-value, field access, and bool default
- [ ] Verify `go test ./internal/types/...` passes with 100% coverage (AC: 5, 6)
- [ ] Verify TypeScript files still present (AC: 7)

## Dev Notes

### Module Initialization

```bash
# Run from repo root (terminus.platform local path)
go mod init github.com/electricm0nk/terminus-platform
```

Go version in `go.mod` must be exactly `go 1.25`. Do NOT use `go 1.25.0` — use the two-part form.

### Type Definitions

**`internal/types/module.go`**

Semantic equivalent of the existing TypeScript `ModulePayload` and `RunContext` types from `shared/`:

```go
package types

// RunContext carries runtime metadata passed to module execution.
type RunContext struct {
    UserID    string
    SessionID string
    Timestamp string
}

// ModulePayload is a generic payload container for workflow inputs.
type ModulePayload[S any] struct {
    Schema  S
    Context RunContext
}

// Module is the interface all workflow modules must satisfy.
type Module interface {
    Name() string
}
```

**`internal/types/release.go`**

```go
package types

// ReleaseInput is the workflow input for ReleaseWorkflow.
type ReleaseInput struct {
    Service     string
    SHA         string
    Env         string
    ProvisionDB bool
}

// ReleaseResult is the workflow output for ReleaseWorkflow.
type ReleaseResult struct {
    Status          string
    CompletedAt     string
    SmokeTestPassed bool
}
```

### TDD Requirement

Test files must be created BEFORE the implementation files. Tests cover:
- Zero-value construction (`var r ReleaseInput` — all fields at zero value)
- Field assignment and retrieval
- `ProvisionDB` default is `false`
- `SmokeTestPassed` default is `false`
- `ModulePayload[T]` with a concrete type parameter

### No External Dependencies at This Stage

`go.sum` may be empty. Do not add `go.temporal.io/sdk` yet — that happens in Story 1.2 when the worker entrypoint is wired.

### Coexistence with TypeScript

The TypeScript workspace (`services/`, `shared/`, `package.json`, `tsconfig.json`) MUST remain untouched in this story. The Go module lives alongside it. Removal happens in Story 1.4 after CI is updated.

### Project Structure After This Story

```
terminus.platform/
├── go.mod                    # NEW: module github.com/electricm0nk/terminus-platform, go 1.25
├── go.sum                    # NEW: empty
├── internal/
│   └── types/
│       ├── module.go         # NEW
│       ├── module_test.go    # NEW (TDD: written first)
│       ├── release.go        # NEW
│       └── release_test.go   # NEW (TDD: written first)
├── services/                 # EXISTING: TypeScript — do not touch
├── shared/                   # EXISTING: TypeScript — do not touch
├── package.json              # EXISTING — do not touch
└── ...
```

### References

- [Architecture: Language Decision](docs/terminus/platform/releaseorchestrator/architecture.md#project-context)
- [Architecture: Data Architecture](docs/terminus/platform/releaseorchestrator/architecture.md#data-architecture)
- [Architecture: Naming Patterns](docs/terminus/platform/releaseorchestrator/architecture.md#naming-patterns)
- [Architecture: Mandatory Rules](docs/terminus/platform/releaseorchestrator/architecture.md#mandatory-rules)
- [Epics: Story 1.1](docs/terminus/platform/releaseorchestrator/epics.md#story-11-go-module-scaffold-and-type-foundation)

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List
