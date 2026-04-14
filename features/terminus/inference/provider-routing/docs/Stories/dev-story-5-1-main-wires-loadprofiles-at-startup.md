# Story 5.1: main.go wires LoadProfiles at startup

**Initiative:** terminus-inference-provider-routing
**Epic:** 5 — Gateway can deploy with routing enabled
**Status:** review

## Story

As a **gateway developer**,
I want `main.go` to call `LoadProfiles` at startup and inject the returned `Router` into the server,
So that the routing layer is active for every request from process start.

## Acceptance Criteria

**Given** `ROUTING_CONFIG_PATH` is set and points to a valid config file
**When** the gateway starts
**Then** `routing.LoadProfiles(os.Getenv("ROUTING_CONFIG_PATH"))` is called before the HTTP server begins accepting requests
**And** if `LoadProfiles` returns an error, the gateway logs the error and exits with a non-zero code
**And** the returned `Router` is injected into the server struct — no global variable

**Given** `ROUTING_CONFIG_PATH` is not set
**When** the gateway starts
**Then** it exits immediately with `"routing: ROUTING_CONFIG_PATH environment variable is not set"`

## Technical Notes

- Target repo: `terminus-inference-gateway`
- File: `cmd/gateway/main.go` (or wherever current main lives)
- Pattern:
  ```go
  router, err := routing.LoadProfiles(os.Getenv("ROUTING_CONFIG_PATH"))
  if err != nil {
      slog.Error("routing config failed", "error", err)
      os.Exit(1)
  }
  srv := server.New(cfg, router)  // inject, not global
  ```
- The server struct must accept `routing.Router` interface (not concrete type) to remain testable
- `ROUTING_CONFIG_PATH` is the only injection point — no fallback path

## Dependencies

- Epic 1 complete (LoadProfiles implemented and tested)
- Epic 2 complete (Router interface finalized)
