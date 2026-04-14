# Story 5.1: main.go wires LoadProfiles at startup

Status: ready-for-dev

## Story

As a **gateway developer**,
I want `main.go` to call `LoadProfiles` at startup and inject the returned `Router` into the server,
So that the routing layer is active for every request from process start.

## Acceptance Criteria

1. **Given** `ROUTING_CONFIG_PATH` is set and points to a valid config file  
   **When** the gateway starts  
   **Then** `routing.LoadProfiles(os.Getenv("ROUTING_CONFIG_PATH"))` is called before the HTTP server begins accepting requests  
   **And** if `LoadProfiles` returns an error, the gateway logs the error and exits with a non-zero code  
   **And** the returned `Router` is injected into the server struct — no global variable

2. **Given** `ROUTING_CONFIG_PATH` is not set  
   **When** the gateway starts  
   **Then** it exits immediately with `"routing: ROUTING_CONFIG_PATH environment variable is not set"`

## Tasks / Subtasks

- [ ] Add `routing.LoadProfiles(os.Getenv("ROUTING_CONFIG_PATH"))` call in `main.go` startup sequence (AC: 1)
  - [ ] Must occur BEFORE `http.ListenAndServe` (or equivalent server start) (AC: 1)
  - [ ] On error: `log.Fatalf("routing: LoadProfiles: %v", err)` or `slog.Error` + `os.Exit(1)` (AC: 1)
  - [ ] Assign returned `Router` to a local variable; inject into server via constructor/field — no `var globalRouter` (AC: 1)
- [ ] Propagate `Router` to the server struct field (AC: 1)
  - [ ] Locate existing server struct; add `router routing.Router` field
  - [ ] Update server constructor to accept `routing.Router` parameter
- [ ] Verify `ROUTING_CONFIG_PATH` unset path exits with correct message (AC: 2)
- [ ] Verify `go build ./...` passes

## Dev Notes

- This story touches `main.go` and likely a `server` or `app` struct in the gateway codebase
- Gateway architecture constraint: `Router` interface is the sole integration boundary (NFR-I1) — `main.go` imports `internal/routing` and calls `LoadProfiles`, then passes the result; no other file in the gateway should import `internal/routing` except handlers (Story 5.2)
- `os.Getenv("ROUTING_CONFIG_PATH")` read in `main.go`, passed to `LoadProfiles` — the function itself does not read env vars (established in Story 1.2)
- Startup failure is a hard exit — `log.Fatalf` or `slog.Error + os.Exit(1)` both acceptable; prefer `slog.Error` for consistency with gateway logging
- No global Router variable — thread-safety and testability both depend on dependency injection

### Project Structure Notes

- File: `main.go` (gateway entrypoint) + server struct file (usually `server.go` or `cmd/gateway/main.go`)
- Import: `"github.com/..../terminus-inference-gateway/internal/routing"` — confirm module path from `go.mod`
- Check existing gateway startup sequence for where to insert the `LoadProfiles` call (after flag/env parsing, before HTTP server start)

### References

- [Source: docs/terminus/inference/provider-routing/prd.md#FR1] — Gateway calls Resolve on every request (requires Router injection)
- [Source: docs/terminus/inference/provider-routing/architecture.md#Additional Requirements] — `main.go` wiring; `ROUTING_CONFIG_PATH`; no global variable
- [Source: docs/terminus/inference/provider-routing/epics.md#Story 5.1] — All AC scenarios

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
