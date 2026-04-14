---
id: "1-1"
epic: 1
story: 1
title: "Add repository_dispatch CI handler to fourdogs-central"
status: in-progress
target_repo: fourdogs-central
target_path: TargetProjects/fourdogs/central/fourdogs-central
---

# Story 1-1: Add repository_dispatch CI handler to fourdogs-central

**Epic:** 1 — Fix missing UI in production deployment
**Story:** 1-1
**Status:** in-progress

## Problem

`fourdogs-central/.github/workflows/ci.yml` has no handler for the `repository_dispatch`
event `ui-build-complete` fired by `fourdogs-central-ui` CI on every successful main push.
Every Docker image shipped to date embeds the placeholder `cmd/api/dist/index.html`.

## User Story

As a **user of central.fourdogspetsupplies.com**,
I want the React SPA served instead of the placeholder HTML,
So that I can use the actual Four Dogs Pet Supplies application.

## Acceptance Criteria

- [ ] `fourdogs-central/.github/workflows/ci.yml` handles `repository_dispatch: types: [ui-build-complete]`
- [ ] New job `build-with-ui` checks out `fourdogs-central-ui` at the dispatched SHA
- [ ] `dist/` contents are copied from `fourdogs-central-ui/dist/` to `cmd/api/dist/` before `go build`
- [ ] Docker image is built with the real SPA embedded (not the placeholder)
- [ ] Docker image is tagged with the UI repo commit SHA from `client_payload.sha`
- [ ] Helm `values.yaml` image tag is updated and committed `[skip ci]`
- [ ] Existing `build-test` and `docker-publish` jobs are unchanged
- [ ] `build-with-ui` does NOT run on push/PR triggers (only on `repository_dispatch`)

## Tasks

1. Add `repository_dispatch: types: [ui-build-complete]` to existing `on:` block
2. Add `build-with-ui` job:
   - `if: github.event_name == 'repository_dispatch'`
   - Step: checkout `fourdogs-central` (default `main`)
   - Step: checkout `fourdogs-central-ui` at `github.event.client_payload.sha` via `CROSS_REPO_PAT`
   - Step: `cp -r fourdogs-central-ui/dist/. cmd/api/dist/`
   - Step: setup-go (go-version-file: go.mod)
   - Step: `go build ./...`
   - Step: `go test ./... -race -count=1`
   - Step: docker/login-action → GHCR
   - Step: docker/build-push-action, tag = `github.event.client_payload.sha`
   - Step: sed Helm values.yaml tag, git commit `[skip ci]`, git push

## Technical Notes

- `CROSS_REPO_PAT` secret already exists (used by `fourdogs-central-ui` CI for dispatch)
- Placeholder `cmd/api/dist/index.html` stays in repo — required by `go:embed` for non-dispatch builds
- `cp -r` with trailing slash: `fourdogs-central-ui/dist/.` copies contents (not the folder)
- The `build-with-ui` job must NOT have `needs: build-test` — it is independent, dispatch-only
- Git config required before Helm tag commit: `git config user.name "github-actions[bot]"`
