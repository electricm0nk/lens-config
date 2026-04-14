# Story 1.2: Add sqlc Queries for `authorized_users`

Status: ready-for-dev

## Story

As a developer,
I want sqlc queries for all CRUD operations on the `authorized_users` table,
so that the OAuth callback handler and users CLI have type-safe, compile-verified DB operations.

## Acceptance Criteria

1. `internal/db/queries/authorized_users.sql` exists with all four named queries:
   - `IsAuthorized`: `SELECT EXISTS(SELECT 1 FROM authorized_users WHERE google_sub = $1)` → `:one` returning `bool`
   - `AddUser`: `INSERT INTO authorized_users (google_sub, email) VALUES ($1, $2) ON CONFLICT DO NOTHING` → `:exec`
   - `RemoveUser`: `DELETE FROM authorized_users WHERE google_sub = $1` → `:exec`
   - `ListUsers`: `SELECT google_sub, email, added_at FROM authorized_users ORDER BY added_at` → `:many`
2. `sqlc generate` runs without errors and produces `internal/db/authorized_users.sql.go` with correct Go types
3. `go build ./...` passes cleanly after sqlc generation
4. `AddUser` is strictly idempotent: running it twice with the same `google_sub` produces no error and no duplicate row (enforced by `ON CONFLICT DO NOTHING`)

## Tasks / Subtasks

- [ ] Task 0: Verify Story 1.1 complete — `authorized_users` table exists in local dev DB (AC: all)
  - [ ] `psql $DATABASE_URL -c "\d authorized_users"` — confirm table and columns
  - [ ] `make migrate-local` if table is missing

- [ ] Task 1: Write SQL query file (AC: 1)
  - [ ] Create `internal/db/queries/authorized_users.sql`:
    ```sql
    -- name: IsAuthorized :one
    SELECT EXISTS(
        SELECT 1 FROM authorized_users WHERE google_sub = $1
    );

    -- name: AddUser :exec
    INSERT INTO authorized_users (google_sub, email)
    VALUES ($1, $2)
    ON CONFLICT DO NOTHING;

    -- name: RemoveUser :exec
    DELETE FROM authorized_users WHERE google_sub = $1;

    -- name: ListUsers :many
    SELECT google_sub, email, added_at FROM authorized_users ORDER BY added_at;
    ```
  - [ ] Commit: `feat(db): add authorized_users sqlc query file`

- [ ] Task 2: Run sqlc generate and commit output (AC: 2)
  - [ ] `make generate` (or `sqlc generate`)
  - [ ] Confirm `internal/db/authorized_users.sql.go` generated with:
    - `IsAuthorized(ctx, googleSub string) (bool, error)`
    - `AddUser(ctx, arg AddUserParams) error`
    - `RemoveUser(ctx, googleSub string) error`
    - `ListUsers(ctx) ([]AuthorizedUser, error)`
  - [ ] `git add internal/db/authorized_users.sql.go && git commit -m "feat(db): add generated sqlc output for authorized_users"`

- [ ] Task 3: Verify build and idempotency (AC: 3, 4)
  - [ ] `go build ./...` — no errors
  - [ ] Manually test idempotency via psql:
    ```sql
    INSERT INTO authorized_users (google_sub, email) VALUES ('test-sub', 'test@example.com') ON CONFLICT DO NOTHING;
    INSERT INTO authorized_users (google_sub, email) VALUES ('test-sub', 'test@example.com') ON CONFLICT DO NOTHING;
    SELECT count(*) FROM authorized_users WHERE google_sub = 'test-sub'; -- expect: 1
    DELETE FROM authorized_users WHERE google_sub = 'test-sub';
    ```

## Dev Notes

- `IsAuthorized` returns `bool` (via `EXISTS`) — the generated Go signature will be `IsAuthorized(ctx context.Context, googleSub string) (bool, error)`. If sqlc generates `interface{}` instead of `bool` due to `EXISTS`, adjust using a named column: `SELECT EXISTS(...) AS authorized`.
- `ON CONFLICT DO NOTHING` is a hard requirement (adversarial review note). Do not use `ON CONFLICT DO UPDATE` — upsert semantics are not wanted here; existing authorized users must not have their email silently overwritten.
- The `sqlc.yaml` queries path is `internal/db/queries/` and output is `internal/db/` — consistent with foundation pattern (`items.sql`, `sessions.sql`).
- The generated `Queries` struct in `internal/db/` is the same struct used by the OAuth callback handler and CLI binary — all share the same `*pgxpool.Pool` wired in at startup.

### Project Structure Notes

- Query files live at `internal/db/queries/` (NOT `db/query/` as the stories.md draft suggested)
- Generated output goes to `internal/db/` — alongside `items.sql.go` and `sessions.sql.go`
- Existing `Queries` struct in `internal/db/db.go` is extended automatically by sqlc — no manual edits to `db.go` needed

### References

- Foundation sqlc pattern: [_bmad-output/implementation-artifacts/fourdogs-central/dev-story-1-3-sqlc-configuration-and-codegen-ci-gate.md](_bmad-output/implementation-artifacts/fourdogs-central/dev-story-1-3-sqlc-configuration-and-codegen-ci-gate.md)
- Architecture: [_bmad-output/lens-work/initiatives/fourdogs/central/centralauthorization/architecture.md](_bmad-output/lens-work/initiatives/fourdogs/central/centralauthorization/architecture.md)
- Stories source: [_bmad-output/lens-work/initiatives/fourdogs/central/centralauthorization/stories.md](_bmad-output/lens-work/initiatives/fourdogs/central/centralauthorization/stories.md)
- Adversarial review (ON CONFLICT note): [_bmad-output/lens-work/initiatives/fourdogs/central/centralauthorization/adversarial-review-report.md](_bmad-output/lens-work/initiatives/fourdogs/central/centralauthorization/adversarial-review-report.md)

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
