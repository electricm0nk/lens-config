# Story 1.4: TypeScript Scaffold Removal

Status: ready-for-dev

## Story

As a platform developer,
I want the TypeScript/Node.js scaffold fully removed from the repository so that the codebase is exclusively Go and there is no confusion about what runtime the worker uses,
so that future contributors do not accidentally depend on or maintain dead code.

## Acceptance Criteria

1. `services/` directory is deleted in its entirety
2. `shared/` directory is deleted in its entirety (if present)
3. `node_modules/` directory is deleted (if present)
4. `coverage/` directory is deleted (if present)
5. `package.json` is deleted
6. `package-lock.json` is deleted (if present)
7. `tsconfig.json` is deleted (if present)
8. `.npmrc` is deleted (if present)
9. `go build ./...` still passes after removals
10. `go test ./...` still passes after removals
11. No TypeScript/Node.js import or reference in any Go file

**Pre-condition:** Story 1.3 is merged AND the new Go Dockerfile is in place.

**Critical merge order:** This story MUST NOT be merged before Story 1.3 is merged. The new Dockerfile must be live before the old one (in `services/temporal/`) is deleted.

## Tasks / Subtasks

- [ ] Confirm Story 1.3 is merged (pre-condition gate) — do NOT proceed if it isn't
- [ ] Inventory all TypeScript/Node artifacts in the repo (AC: 1–8)
  - [ ] `ls services/` — document what's there
  - [ ] `ls shared/` — document what's there (may not exist)
  - [ ] `ls node_modules/` — document what's there (may not exist)
  - [ ] `ls coverage/` — document what's there (may not exist)
  - [ ] Check for `package.json`, `package-lock.json`, `tsconfig.json`, `.npmrc` at repo root and subdirs
- [ ] Delete all TypeScript/Node artifacts (AC: 1–8)
  - [ ] `rm -rf services/`
  - [ ] `rm -rf shared/` (if present)
  - [ ] `rm -rf node_modules/` (if present)
  - [ ] `rm -rf coverage/` (if present)
  - [ ] `rm -f package.json package-lock.json tsconfig.json .npmrc` (as applicable)
- [ ] Verify Go build still passes: `go build ./...` (AC: 9)
- [ ] Verify Go tests still pass: `go test ./...` (AC: 10)
- [ ] Scan for any Node/TypeScript references in Go files: `grep -r "node_modules\|\.ts\|require(" --include="*.go" .` (AC: 11)
- [ ] Update `.gitignore` — remove Node.js patterns if present; add Go patterns if missing

## Dev Notes

### Why This Is a Separate Story

Separation of Story 1.3 (add Go Dockerfile) from Story 1.4 (delete TypeScript scaffold) ensures the following:
- CI never has a window where there is no valid Dockerfile
- If Story 1.3 has a problem discovered post-merge, Stories 1.4–2.x can be blocked while it's fixed
- Git history has a clean "added Go Dockerfile" commit separate from "removed TypeScript"

This was mandated by Adversarial Review Finding 3 (migration atomicity).

### What to Delete vs. What to Keep

**Delete everything under `services/`** — this directory contains the Node.js Temporal worker. After this story, all Temporal worker code is under `cmd/worker/` and `internal/`.

**Delete `shared/`** — TypeScript shared utilities, now replaced by `internal/types/`.

**Delete Node.js toolchain files** — `package.json`, `package-lock.json`, `tsconfig.json`, `.npmrc` (if any exist at root or subdirectory level).

**Keep everything under (already created):**
- `cmd/`
- `internal/`
- `k8s/`
- `helm/`
- `Dockerfile` (new Go multi-stage, created in Story 1.3)
- `go.mod`, `go.sum`
- `docs/`
- `.github/` (CI workflows, but verify no lingering Node steps)

### CI Pipeline Cleanup

After removing `services/`, scan the CI pipeline (`.github/workflows/`) for any steps that reference:
- `npm install`, `npm ci`, `npm run build`
- `node_modules`
- `tsc`

If any such steps exist, delete them. The CI pipeline should now only have:
- `go build ./...`
- `go test ./...`
- Docker build + push (already updated in Story 1.3)

### .gitignore

Remove any Node.js patterns (`node_modules/`, `*.js` if broad, `dist/`, etc.) that are no longer relevant. Add Go patterns if missing:

```gitignore
# Go
/tmp/
*.test
*.out
vendor/
```

### Verification Checklist

```bash
# All should succeed:
go build ./...
go test ./...

# All should be empty / not found:
find . -name "package.json" -not -path "./.git/*"
find . -name "node_modules" -not -path "./.git/*"
find . -name "tsconfig.json" -not -path "./.git/*"
ls services/ 2>&1   # should say "No such file or directory"
```

### References

- [Architecture: Repository Transformation](docs/terminus/platform/releaseorchestrator/architecture.md#repository-transformation)
- [Architecture: Project Structure — After](docs/terminus/platform/releaseorchestrator/architecture.md#go-module-structure)
- [Epics: Story 1.4](docs/terminus/platform/releaseorchestrator/epics.md#story-14-typescript-scaffold-removal)
- Adversarial review Finding 3 (migration atomicity): This story MUST merge after Story 1.3

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List
