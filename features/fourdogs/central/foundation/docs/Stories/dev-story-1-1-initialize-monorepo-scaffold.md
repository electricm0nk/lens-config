# Story 1.1: Initialize Monorepo Scaffold

Status: ready-for-dev

## Story

As Todd (developer),
I want the monorepo initialized with the correct directory structure, Go module, and config layer,
so that all subsequent stories have a consistent, convention-based foundation to build on.

## Acceptance Criteria

1. `go mod init github.com/electricm0nk/fourdogs-central` done; `go.mod` exists with correct module path
2. Directory structure: `cmd/{api,ingest,migrate}/main.go` stubs, `internal/{auth,config,db,handler,ingest}/` package stubs, `migrations/`, `deploy/helm/fourdogs-central/`, `.github/workflows/`
3. `internal/config/config.go` reads env vars into `Config` struct: `DATABASE_URL`, `OAUTH_CLIENT_ID`, `OAUTH_CLIENT_SECRET`, `CORS_ALLOWED_ORIGIN`
4. Startup (`cmd/api/main.go`) fails fast with descriptive error if any required env var is missing
5. `Makefile` has `generate`, `test`, `build`, `migrate-local` targets
6. `go build ./...` passes with no errors on stub scaffold

## Tasks / Subtasks

- [ ] Task 0: Sync target repo to latest main and verify clean state (AC: all)
  - [ ] `cd TargetProjects/fourdogs/central/fourdogs-central && git checkout main && git pull origin main`
  - [ ] Confirm empty / stub state (README only or fresh repo)

- [ ] Task 1: Write failing tests for config layer before implementation (AC: 3, 4) — RED phase
  - [ ] Create `internal/config/config_test.go`:
    - Test: all 4 env vars present → `Config` struct populated correctly
    - Test: `DATABASE_URL` missing → startup returns error containing "DATABASE_URL"
    - Test: `OAUTH_CLIENT_ID` missing → startup returns error containing "OAUTH_CLIENT_ID"
    - Test: `OAUTH_CLIENT_SECRET` missing → startup returns error containing "OAUTH_CLIENT_SECRET"
    - Test: `CORS_ALLOWED_ORIGIN` missing → startup returns error containing "CORS_ALLOWED_ORIGIN"
  - [ ] Run `go test ./internal/config/...` — confirm compilation fails (no implementation yet)

- [ ] Task 2: Initialize Go module and directory skeleton (AC: 1, 2)
  - [ ] `go mod init github.com/electricm0nk/fourdogs-central`
  - [ ] Create directory tree:
    ```
    cmd/
      api/main.go       # stub: package main / func main() {}
      ingest/main.go    # stub
      migrate/main.go   # stub
    internal/
      auth/auth.go      # stub: package auth
      config/config.go  # (implemented in Task 3)
      db/db.go          # stub: package db
      handler/handler.go # stub: package handler
      ingest/ingest.go  # stub: package ingest
    migrations/         # empty dir (gitkeep)
    deploy/
      helm/
        fourdogs-central/  # empty dir (gitkeep)
    .github/
      workflows/        # empty dir (gitkeep)
    ```
  - [ ] Create `.gitignore`:
    ```
    fourdogs-central
    fourdogs-central-api
    fourdogs-central-ingest
    fourdogs-central-migrate
    *.test
    coverage.out
    ```
  - [ ] Commit: `chore(scaffold): initialize go module and directory structure`

- [ ] Task 3: Implement `internal/config/config.go` (AC: 3, 4) — GREEN phase
  - [ ] Create `Config` struct:
    ```go
    type Config struct {
        DatabaseURL       string
        OAuthClientID     string
        OAuthClientSecret string
        CORSAllowedOrigin string
    }
    ```
  - [ ] Implement `Load() (*Config, error)`:
    - Read each env var with `os.Getenv`
    - If any is empty, return `nil, fmt.Errorf("missing required env var: %s", varName)`
    - Return populated `Config`
  - [ ] Run `go test ./internal/config/...` — all tests must pass (GREEN)
  - [ ] Commit: `feat(config): add Config struct with required env var validation`

- [ ] Task 4: Wire `cmd/api/main.go` with config load and fail-fast (AC: 4)
  - [ ] `cmd/api/main.go`:
    ```go
    func main() {
        cfg, err := config.Load()
        if err != nil {
            log.Fatalf("startup error: %v", err)
        }
        _ = cfg // placeholder until router wired in Story 3.1
        log.Println("fourdogs-central api starting...")
    }
    ```
  - [ ] Commit: `feat(api): wire config load with fail-fast on startup`

- [ ] Task 5: Create Makefile (AC: 5)
  - [ ]
    ```makefile
    .PHONY: generate test build migrate-local

    generate:
    	go generate ./...
    	sqlc generate

    test:
    	go test ./... -race -count=1

    build:
    	go build -o bin/api ./cmd/api
    	go build -o bin/ingest ./cmd/ingest
    	go build -o bin/migrate ./cmd/migrate

    migrate-local:
    	DATABASE_URL=$${DATABASE_URL} go run ./cmd/migrate up
    ```
  - [ ] Commit: `chore(makefile): add generate, test, build, migrate-local targets`

- [ ] Task 6: Verify `go build ./...` passes (AC: 6)
  - [ ] Run `go build ./...` — must pass with no errors
  - [ ] Commit: `chore(scaffold): verify clean build on stub scaffold`

- [ ] Task 7: Update sprint status
  - [ ] Set `1-1-initialize-monorepo-scaffold: in-progress` at task start → `done` at completion
  - [ ] Set `epic-1: in-progress`

## Dev Notes

- Target repo: `TargetProjects/fourdogs/central/fourdogs-central`
- Go version: 1.22+ (use go1.22 toolchain directive in go.mod)
- Config must use env vars only — no config files, no `.env` in prod path
- Fail-fast pattern: `log.Fatalf` in main is acceptable; no global state
- `internal/config` should NOT import any other internal packages (avoid circular deps)
- All package stubs need only `package <name>` — no imports, no functions yet

### Project Structure Notes

- Module: `github.com/electricm0nk/fourdogs-central`
- All business logic lives in `internal/` — nothing exported from `cmd/`
- `cmd/{api,ingest,migrate}` are thin entry points only
- ARCH11 sequencing: `cmd/migrate` embeds `migrations/` → all other commands depend on DB being migrated

### References

- [Source: phases/techplan/architecture.md — Directory Layout]
- [Source: phases/devproposal/epics.md — Story 1.1 ACs]
- [Source: phases/businessplan/prd.md — FR25 (versioned routes), FR26 (k3s deployment)]

## Dev Agent Record

### Agent Model Used

GitHub Copilot / Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
