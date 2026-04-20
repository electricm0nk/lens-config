# Story: transactions-1-04

## Context

**Feature:** transactions
**Sprint:** 1 — Core Implementation
**Priority:** medium
**Estimate:** S

## User Story

As a developer, I want the GitHub Actions CI pipeline to run `go test` and `go vet` for the new
`internal/etailpet/` package and `cmd/etailpet-trigger` binary, so that regressions are caught
automatically on every push.

## Acceptance Criteria

- [ ] The existing `.github/workflows/` CI workflow in `fourdogs-central` repo includes `./internal/etailpet/...` and `./cmd/etailpet-trigger/...` in its test and vet targets (or the existing `./...` pattern already covers them — confirm)
- [ ] CI runs `go test ./... -count=1` and `go vet ./...` — both pass on `transactions` branch
- [ ] No new workflow file required if existing `./...` glob already covers the new packages
- [ ] If a new workflow step is required (e.g. for separate binary image build), it is added to the existing workflow file only — no new workflow files
- [ ] CI passes on a PR from `transactions` to `transactions-plan` (dry-run validation)

## Technical Notes

Check the existing CI workflow in `fourdogs-central`:
```bash
cat .github/workflows/*.yml | grep -E "go test|go vet|go build"
```

If `go test ./...` is already the command, no changes are needed — the new packages are automatically
included. Document the finding in the PR description.

If the workflow runs only specific package paths, add the new paths:
```yaml
- run: go test ./internal/etailpet/... ./cmd/etailpet-trigger/... -count=1
- run: go vet ./internal/etailpet/... ./cmd/etailpet-trigger/...
```

The binary Docker image build for `fourdogs-etailpet-trigger` is handled in story transactions-2-01
(Helm chart + ArgoCD). This story is only about the test/vet gate.

## Dependencies

**Blocked by:** transactions-1-01, transactions-1-02 (packages must exist for CI to test)
**Blocks:** nothing

## Definition of Done

- [ ] Code compiles with no errors or warnings (`go build ./...`)
- [ ] `go test ./...` passes
- [ ] `go vet ./...` passes
- [ ] No credentials, tokens, or secrets in code or committed files
- [ ] Structured log events fire correctly for the story's scope
- [ ] Story acceptance criteria verified by author
- [ ] PR reviewed (or self-reviewed for solo work) and merged to `transactions` branch

## Open Questions

None.
