# Story 1.4: CI Pipeline — Build, Test, Push

Status: done

## Story

As a developer,
I want a GitHub Actions workflow that builds, tests, and pushes the image on merge to main,
so that ArgoCD always pulls a tested, tagged image.

## Acceptance Criteria

1. `npm ci && npm test` must pass before Docker build proceeds
2. Docker image is tagged with git SHA and pushed to `ghcr.io/electricm0nk/terminus-portal`
3. `latest` tag is also updated on successful main merge
4. Branch protection prevents merge if tests fail
5. Workflow file lives at `.github/workflows/ci.yml`

## Tasks / Subtasks

- [ ] Task 1: Create `.github/workflows/ci.yml` (AC: #5)
  - [ ] Trigger on: `push` to `main`, `pull_request` targeting `main`
  - [ ] Job 1 `test`: `node:20`, `npm ci`, `npm test`
  - [ ] Job 2 `build-push`: depends on `test`, only on `main` push
    - [ ] Login to GHCR: `docker/login-action` with `secrets.GITHUB_TOKEN`
    - [ ] Build + push with SHA tag: `ghcr.io/electricm0nk/terminus-portal:${{ github.sha }}`
    - [ ] Also tag + push `latest` on main push (AC: #3)
- [ ] Task 2: Set `permissions: packages: write` in workflow (AC: #2)
  - [ ] Required for GHCR push with `GITHUB_TOKEN`
- [ ] Task 3: Document branch protection note (AC: #4)
  - [ ] Add comment in `ci.yml` header: branch protection is configured separately in GitHub repo settings (Settings → Branches → Require status checks: `test`)
- [ ] Task 4: Verify workflow syntax (AC: #5)
  - [ ] `yamllint .github/workflows/ci.yml` (or equivalent)
  - [ ] Optionally: `act --dry-run` for local validation if `act` is available

## Dev Notes

**Target repo:** `TargetProjects/terminus/portal/terminus-portal`
**Registry:** `ghcr.io/electricm0nk/terminus-portal` — authenticated via `GITHUB_TOKEN` (no PAT needed for same-org pushes)
**Workflow structure:**

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read
  packages: write

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm test

  build-push:
    needs: test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: |
            ghcr.io/electricm0nk/terminus-portal:${{ github.sha }}
            ghcr.io/electricm0nk/terminus-portal:latest
```

### Project Structure Notes

New file only:
```
terminus-portal/
└── .github/
    └── workflows/
        └── ci.yml
```

### References

- NFR1 (TDD — tests before build): `docs/terminus/portal/portal2/epics.md`
- NFR3 (ArgoCD pulls tested image): `docs/terminus/portal/portal2/epics.md`
- Story 1.2 (Dockerfile prerequisite), Story 1.3 (Helm chart context)

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List
