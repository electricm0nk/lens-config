# Story: sales-1-04

## Context

**Feature:** fourdogs-etailpet-api-sales
**Sprint:** 1 — LegacyClient and Sales Trigger Binary
**Priority:** medium
**Estimate:** S

## User Story

As a developer, I want the `go test` and `go vet` CI steps to cover the new `internal/etailpet/`
legacy package and `cmd/etailpet-sales-trigger` binary, so that regressions in both the sales
worker and the existing transactions worker are caught automatically.

## Acceptance Criteria

- [ ] `go test ./...` in CI covers `internal/etailpet/` (including legacy client tests) and `cmd/etailpet-sales-trigger/`
- [ ] `go vet ./...` covers the same packages
- [ ] Existing transactions worker CI steps remain green and unaffected
- [ ] CI runs on pull requests targeting `fourdogs-etailpet-api-sales` and `main` branches
- [ ] No test in CI makes network calls (mock transport enforced; confirmed by running `go test -run ./... -count=1` in CI without network access)

## Technical Notes

The CI workflow already runs `go test ./...` and `go vet ./...` for the transactions feature.
Since both workers live in the same `fourdogs-central` repo and Go module, no new CI workflow is
required — verify that the existing job scope covers the new packages.

**Verification steps:**
1. Confirm the CI matrix or `go test ./...` glob covers `internal/etailpet/` (it should, since it
   uses the full `./...` wildcard)
2. If the CI job is scoped to specific packages or directories, add `./internal/etailpet/...` and
   `./cmd/etailpet-sales-trigger/...` explicitly
3. Confirm `go build ./cmd/etailpet-sales-trigger/...` is not blocked by missing dependencies

This is a verification story, not a new workflow story. If CI already covers the new packages
via `./...`, document that and close with the evidence.

## Dependencies

**Blocked by:** sales-1-01, sales-1-02 (new packages must exist before CI can verify them)
**Blocks:** none

## Definition of Done

- [ ] CI job passes with new packages included
- [ ] Transactions worker CI unaffected (confirm green)
- [ ] Story acceptance criteria verified by author
- [ ] PR reviewed and merged to `fourdogs-etailpet-api-sales` branch
