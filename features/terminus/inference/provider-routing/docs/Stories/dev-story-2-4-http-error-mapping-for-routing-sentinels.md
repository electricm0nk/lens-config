# Story 2.4: HTTP error mapping for routing sentinel errors

**Initiative:** terminus-inference-provider-routing
**Epic:** 2 — Gateway can route requests to the correct provider
**Status:** review

## Story

As a **gateway developer**,
I want `internal/errors` to map routing sentinel errors to the correct HTTP status codes,
So that callers receive 503 for no-eligible-provider and 400 for invalid-workload-class without custom handler logic.

## Acceptance Criteria

**Given** an error where `errors.Is(err, routing.ErrNoEligibleProvider)` is true
**When** it reaches the gateway's error mapping layer in `internal/errors`
**Then** it maps to `HTTP 503 Service Unavailable`

**Given** an error where `errors.Is(err, routing.ErrInvalidWorkloadClass)` is true
**When** it reaches the gateway's error mapping layer
**Then** it maps to `HTTP 400 Bad Request`

**Given** a wrapped error `fmt.Errorf("routing: no eligible provider for profile %q: %w", profile, ErrNoEligibleProvider)`
**When** `errors.Is(err, routing.ErrNoEligibleProvider)` is evaluated
**Then** it returns `true` (wrapping must not break sentinel matching)

## Technical Notes

- Target repo: `terminus-inference-gateway`
- File: `internal/errors/` (existing package — add routing sentinel entries to existing HTTP mapping)
- Use `errors.Is` chain matching — do NOT compare error strings
- The existing error mapper likely has a switch or map structure; add two new cases
- Import `internal/routing` from `internal/errors` — check for circular import risk (routing must not import internal/errors)

## Dependencies

- Story 2.3 (sentinel errors defined and tested)
