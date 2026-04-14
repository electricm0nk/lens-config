# Story 1.4: Add `users` CLI Subcommand

Status: ready-for-dev

## Story

As a system operator,
I want a `users` CLI binary with `add`, `remove`, and `list` subcommands,
so that I can manage the `authorized_users` table directly from the server without a UI or raw SQL.

## Acceptance Criteria

1. `cmd/users/main.go` exists and compiles
2. `users add <google_sub> <email>` — inserts the record (idempotent via `ON CONFLICT DO NOTHING`); prints `✅ Added <email> (<google_sub>)` on success
3. `users remove <google_sub>` — deletes the record; prints `✅ Removed <google_sub>` on success; prints a clear error message if the sub is not found (non-zero exit)
4. `users list` — prints a formatted table of `google_sub | email | added_at`; on empty table, prints a clear empty-state message (not an error, exit code 0)
5. Uses `DATABASE_URL` environment variable for the pgxpool connection (same pattern as ingestion CLI at `cmd/ingest/main.go`)
6. `go build ./cmd/users/` produces a working binary named `fourdogs-central-users` (or `users` — match the naming convention of other binaries in the repo)
7. Dockerfile multi-stage build includes a build stage and `COPY` step for this binary, following the same pattern as the ingestion CLI

## Tasks / Subtasks

- [ ] Task 0: Verify Story 1.2 complete — `queries.AddUser`, `RemoveUser`, `ListUsers` available (AC: all)
  - [ ] `go build ./...` passes; confirm `internal/db/authorized_users.sql.go` exists
  - [ ] Review `cmd/ingest/main.go` for pgxpool connection pattern to replicate exactly

- [ ] Task 1: Write failing tests (AC: 2, 3, 4) — RED phase
  - [ ] If the CLI is structured with a testable `run(args, db)` function:
    - Test: `add <sub> <email>` → calls `AddUser`, prints expected output
    - Test: `remove <sub>` → calls `RemoveUser`, prints expected output
    - Test: `remove <sub>` when sub not found → non-zero exit, error message
    - Test: `list` on empty table → exit code 0, empty-state message (not empty output/error)
  - [ ] Confirm tests fail (RED — no implementation yet)

- [ ] Task 2: Implement `cmd/users/main.go` (AC: 1–5)
  - [ ] Connect to Postgres using `pgxpool.New(ctx, os.Getenv("DATABASE_URL"))` — fail fast if `DATABASE_URL` empty
  - [ ] Instantiate `db.New(pool)` for `*db.Queries`
  - [ ] Parse `os.Args[1]` as subcommand:
    - `add`: require 2 args (`google_sub`, `email`); call `queries.AddUser(ctx, db.AddUserParams{GoogleSub: sub, Email: email})`; print `✅ Added <email> (<sub>)`
    - `remove`: require 1 arg (`google_sub`); call `queries.RemoveUser(ctx, sub)`; verify affected row (if sqlc `:exec` — may need explicit check or use `RemoveUser` with a `SELECT EXISTS` pre-check); print `✅ Removed <sub>` or error if not found
    - `list`: call `queries.ListUsers(ctx)`; if empty, print `No authorized users.`; else print formatted table:
      ```
      GOOGLE_SUB                         EMAIL                        ADDED_AT
      ──────────────────────────────────────────────────────────────────────────
      <sub>                              <email>                      <date>
      ```
  - [ ] On missing `DATABASE_URL` or invalid subcommand: print usage, exit 1
  - [ ] Commit: `feat(users): add users CLI binary with add/remove/list subcommands`

- [ ] Task 3: Run tests (AC: 2, 3, 4) — GREEN phase
  - [ ] `go test ./cmd/users/...` — all tests pass
  - [ ] Manual smoke test using local Postgres:
    - `DATABASE_URL=$DATABASE_URL go run ./cmd/users add sub123 betsy@example.com`
    - `DATABASE_URL=$DATABASE_URL go run ./cmd/users add sub123 betsy@example.com` (again — no error, no duplicate)
    - `DATABASE_URL=$DATABASE_URL go run ./cmd/users list`
    - `DATABASE_URL=$DATABASE_URL go run ./cmd/users remove sub123`
    - `DATABASE_URL=$DATABASE_URL go run ./cmd/users list` (expect empty-state message)

- [ ] Task 4: Update Dockerfile (AC: 7)
  - [ ] Locate the multi-stage Dockerfile (likely at repo root or `deploy/`)
  - [ ] Add a build stage for `cmd/users/main.go` following the same pattern as the ingest binary
  - [ ] Add a `COPY --from=builder` step to include the binary in the final image
  - [ ] Binary name: match the naming convention — likely `fourdogs-central-users`
  - [ ] `docker build .` — confirm image builds cleanly
  - [ ] Commit: `feat(users): add users binary to Dockerfile multi-stage build`

- [ ] Task 5: Final build and coverage check (AC: 6)
  - [ ] `go build ./cmd/users/` — produces binary without error
  - [ ] `go test ./...` — no regressions

## Dev Notes

- The `remove` subcommand: sqlc's `:exec` annotation does not return a row count. Options:
  1. Run `SELECT EXISTS` before delete and error if not found
  2. Use `pgxpool` directly with `ExecContext` and check `RowsAffected()` — but this bypasses sqlc type safety
  3. Preferred: add a `GetUser` query (`:one`, returns error if not found) as a pre-check, then delete
  Note: A new `GetUser` query in `authorized_users.sql` + re-run `sqlc generate` is acceptable for this story. Document in the file list.
- `users list` with empty result: `queries.ListUsers` returns `[]db.AuthorizedUser, nil` on empty table — `nil` error, empty slice. Do **not** treat this as an error. Print `No authorized users.` and exit 0.
- pgxpool connection pattern from `cmd/ingest/main.go`: use `pgxpool.New(ctx, dsn)` where `dsn = os.Getenv("DATABASE_URL")`. Defer `pool.Close()`. This is the established foundation pattern.
- Binary naming: check existing binaries in the multi-stage Dockerfile — the API binary is likely `fourdogs-central` and the ingest binary is `fourdogs-central-ingest`. Follow the same suffix pattern: `fourdogs-central-users`.

### Project Structure Notes

- New file: `cmd/users/main.go`
- No new packages — reuse `internal/db` (Queries) and standard library only
- The `cmd/` directory already has `api/`, `ingest/`, `migrate/` — `users/` follows the same pattern
- Dockerfile is at repo root (confirmed by typical Go project layout from Story 1.5 Helm/ArgoCD deployment)

### References

- Ingestion CLI pattern: [_bmad-output/implementation-artifacts/fourdogs-central/dev-story-2-1-etailpet-product-csv-ingestion-parse-and-validate.md](_bmad-output/implementation-artifacts/fourdogs-central/dev-story-2-1-etailpet-product-csv-ingestion-parse-and-validate.md)
- Foundation scaffolding (cmd/ structure): [_bmad-output/implementation-artifacts/fourdogs-central/dev-story-1-1-initialize-monorepo-scaffold.md](_bmad-output/implementation-artifacts/fourdogs-central/dev-story-1-1-initialize-monorepo-scaffold.md)
- Architecture: [_bmad-output/lens-work/initiatives/fourdogs/central/centralauthorization/architecture.md](_bmad-output/lens-work/initiatives/fourdogs/central/centralauthorization/architecture.md)
- Stories source: [_bmad-output/lens-work/initiatives/fourdogs/central/centralauthorization/stories.md](_bmad-output/lens-work/initiatives/fourdogs/central/centralauthorization/stories.md)

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
