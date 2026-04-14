---
stepsCompleted: [1, 2, 3, 4]
lastStep: 4
status: complete
completedAt: '2026-04-13'
inputDocuments:
  - docs/fourdogs/central/central-ui/architecture.md
  - docs/fourdogs/central/central-ui/Stories/1-4-cicd-pipeline-and-embedfs-production-serving.md
  - TargetProjects/fourdogs/central/fourdogs-central/.github/workflows/ci.yml
  - TargetProjects/fourdogs/central/fourdogs-central-ui/.github/workflows/ci.yml
  - TargetProjects/fourdogs/central/fourdogs-central/cmd/api/main.go
  - TargetProjects/fourdogs/central/fourdogs-central/cmd/api/spa_handler.go
  - TargetProjects/fourdogs/central/fourdogs-central/cmd/api/dist/index.html
  - TargetProjects/fourdogs/central/fourdogs-central/Dockerfile
workflowType: architecture
project_name: fourdogs-central-uifix
initiative_root: fourdogs-central-uifix
track: hotfix
---

# Architecture Decision Document — fourdogs-central-uifix

**Author:** Todd Hintzmann
**Date:** 2026-04-13
**Track:** hotfix
**Phase:** techplan

---

## 1. Problem Statement

`central.fourdogspetsupplies.com` is serving the placeholder HTML file committed to
`fourdogs-central/cmd/api/dist/index.html` instead of the built React SPA from
`fourdogs-central-ui`.

**Observed symptom:**
```
Build output placeholder — replaced by CI from fourdogs-central-ui dist/
```

---

## 2. Root Cause Analysis

### System Design (as intended)

```
fourdogs-central-ui push (main)
  → CI: npm ci → build → test → tsc
  → dispatch: ui-build-complete {sha}
      ↓
fourdogs-central CI (repository_dispatch handler)
  → checkout fourdogs-central
  → checkout fourdogs-central-ui @ {sha}
  → cp fourdogs-central-ui/dist/* cmd/api/dist/
  → go build (embeds real dist via go:embed)
  → docker build + push → ghcr.io/electricm0nk/fourdogs-central:{sha}
  → helm values.yaml tag update + push
```

### Actual state

`fourdogs-central/.github/workflows/ci.yml` has **no `repository_dispatch` trigger**.
The only triggers are `push: branches: [main]` and `pull_request: branches: [main]`.

When `fourdogs-central-ui` dispatches `ui-build-complete`, the event is received by
GitHub but silently dropped — no job runs.

The `docker-publish` job in `fourdogs-central` CI runs only on direct pushes to `main`.
It does `COPY . .` in the Dockerfile, which includes `cmd/api/dist/index.html` — the
placeholder. Every Docker image shipped to date embeds only the placeholder.

### Contributing factor

The placeholder is intentionally committed to the repo because `//go:embed dist` requires
that the `dist/` directory and at least one file exist at compile time. The design relies
on CI overwriting it before the Docker build step. Without the dispatch handler, that
overwrite never occurs.

---

## 3. Fix Design

### Scope

One file changes: `fourdogs-central/.github/workflows/ci.yml`.

No changes to:
- Application code (`main.go`, `spa_handler.go`)
- Dockerfile
- Helm charts
- `fourdogs-central-ui` (its CI is correctly implemented)

### New trigger

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  repository_dispatch:
    types: [ui-build-complete]
```

### New job: `build-with-ui`

Runs only on `repository_dispatch` (not on push/PR — those still use the
existing `build-test` + `docker-publish` jobs for backend-only changes).

```
Trigger: repository_dispatch (ui-build-complete)
Steps:
  1. checkout fourdogs-central (default ref: main)
  2. checkout fourdogs-central-ui @ github.event.client_payload.sha
     path: fourdogs-central-ui
     token: CROSS_REPO_PAT
  3. cp -r fourdogs-central-ui/dist/. cmd/api/dist/
  4. setup-go (go-version-file: go.mod)
  5. go build ./...
  6. go test ./... -race -count=1
  7. docker/login-action → GHCR
  8. docker/build-push-action → ghcr.io/electricm0nk/fourdogs-central:{sha}
     sha = github.event.client_payload.sha (UI sha — pinned to matching build)
  9. sed image tag in deploy/helm/fourdogs-central/values.yaml
  10. git commit + push [skip ci]
```

### Tag strategy

Use `client_payload.sha` (the UI repo commit SHA) as the Docker image tag for
`ui-build-complete` builds. This pins each Docker image to the exact UI commit that
built it and prevents ambiguity with backend-only build SHAs.

### Idempotency

The `cp -r` step overwrites `cmd/api/dist/`. The placeholder (`index.html`) is replaced
in-place within the Docker build context. The placeholder remains in the git tree — it is
not modified by this workflow. This is correct: the placeholder is the compile-time
`go:embed` requirement; CI replaces it at image build time.

---

## 4. Decision Log

| # | Decision | Rationale |
|---|----------|-----------|
| D1 | Single new job `build-with-ui`, not merged into `build-test` | `build-test` runs on PR builds which have no UI dist available; mixing would either break PR CI or require conditional logic. Clean separation is safer. |
| D2 | Use UI SHA as Docker image tag | Unambiguous traceability: the deployed image SHA directly identifies the UI commit. |
| D3 | No changes to Dockerfile | `COPY . .` + `go:embed` already works correctly once `cmd/api/dist/` is populated. |
| D4 | Placeholder stays in repo | Required by `go:embed` for backend-only CI builds (PRs, direct main pushes without UI dispatch). Placeholder is the correct compile-time sentinel. |
| D5 | `CROSS_REPO_PAT` token reused for UI checkout | Already used by `fourdogs-central-ui` CI for cross-repo dispatch — same PAT has read access to both repos. |

---

## 5. Implementation Checklist

- [ ] Add `repository_dispatch: types: [ui-build-complete]` trigger to `ci.yml`
- [ ] Add `build-with-ui` job with conditions `if: github.event_name == 'repository_dispatch'`
- [ ] Verify `CROSS_REPO_PAT` secret has `contents: read` on `fourdogs-central-ui`
- [ ] Verify `CROSS_REPO_PAT` has `contents: write` on `fourdogs-central` (for helm tag push)
- [ ] Test: push to `fourdogs-central-ui` main → confirm `ui-build-complete` fires → confirm new job runs → confirm image tag update
- [ ] Verify `GET /` at `central.fourdogspetsupplies.com` returns the React app, not the placeholder

---

## 6. Out of Scope

- Backend-only CI jobs (`build-test`, `docker-publish`) — not modified
- `fourdogs-central-ui` CI — already correctly implemented
- Helm chart values — only the `tag:` field is updated, no chart changes
- Application code — no changes required
