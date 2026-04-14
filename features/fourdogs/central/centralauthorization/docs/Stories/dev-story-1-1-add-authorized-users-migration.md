# Story 1.1: Add `authorized_users` Migration

Status: ready-for-dev

## Story

As a developer,
I want a schema migration that creates the `authorized_users` table,
so that the OAuth callback and users CLI have a persistent store for checking and managing authorized identities.

## Acceptance Criteria

1. `migrations/{N}_create_authorized_users.up.sql` exists and creates the table:
   ```sql
   CREATE TABLE authorized_users (
       google_sub  TEXT PRIMARY KEY,
       email       TEXT NOT NULL,
       added_at    TIMESTAMPTZ NOT NULL DEFAULT now()
   );
   ```
2. `migrations/{N}_create_authorized_users.down.sql` exists and cleanly drops the table:
   ```sql
   DROP TABLE IF EXISTS authorized_users;
   ```
3. `{N}` is one higher than the current highest migration sequence number in `migrations/`
4. `golang-migrate` applies the up migration against the local dev Postgres instance without error
5. Running the up migration a second time is a no-op (idempotent via golang-migrate's schema_migrations tracking)
6. Running the down migration removes the table cleanly
7. `go build ./...` still passes after migration files are added

## Tasks / Subtasks

- [ ] Task 0: Sync target repo and verify clean state (AC: all)
  - [ ] `cd TargetProjects/fourdogs/central/fourdogs-central && git checkout main && git pull origin main`
  - [ ] Check current highest migration: `ls migrations/ | sort | tail -1`
  - [ ] Confirm baseline: foundation has `001_create_items` and `002_create_sessions`; oauthredesign likely added none — next should be `003`. Verify before proceeding.

- [ ] Task 1: Create up migration (AC: 1, 3)
  - [ ] Determine next sequence number N (expected: `003`)
  - [ ] Create `migrations/{N}_create_authorized_users.up.sql`:
    ```sql
    CREATE TABLE authorized_users (
        google_sub  TEXT PRIMARY KEY,
        email       TEXT NOT NULL,
        added_at    TIMESTAMPTZ NOT NULL DEFAULT now()
    );
    ```
  - [ ] Commit: `feat(db): add authorized_users migration up`

- [ ] Task 2: Create down migration (AC: 2)
  - [ ] Create `migrations/{N}_create_authorized_users.down.sql`:
    ```sql
    DROP TABLE IF EXISTS authorized_users;
    ```
  - [ ] Commit: `feat(db): add authorized_users migration down`

- [ ] Task 3: Apply migration and verify (AC: 4, 5, 6)
  - [ ] `make migrate-local` — confirm up migration runs cleanly
  - [ ] `make migrate-local` again — confirm no error (idempotent)
  - [ ] Check table exists: `psql $DATABASE_URL -c "\d authorized_users"` — confirm schema matches AC 1
  - [ ] Apply down: `migrate -path migrations -database $DATABASE_URL down 1` — confirm table dropped
  - [ ] Re-apply up: `make migrate-local` — confirm restore works

- [ ] Task 4: Verify build still passes (AC: 7)
  - [ ] `go build ./...` — no errors

## Dev Notes

- Foundation migration pattern uses 3-digit zero-padded sequence: `001_`, `002_`, so `003_` is the expected next prefix.
- The `cmd/migrate/main.go` uses `//go:embed ../../migrations/*.sql` — new files in `migrations/` are automatically picked up; no changes to the migrate command needed.
- `golang-migrate` tracks applied migrations in the `schema_migrations` table; re-running up is always safe.
- `added_at` uses `DEFAULT now()` — not `GENERATED ALWAYS` — consistent with the `updated_at` pattern in `sessions`.

### Project Structure Notes

- Migration files live at `migrations/` (repo root), NOT `db/migrations/` — the `sqlc.yaml` schema path is `"migrations/"` and the embed path is `../../migrations/*.sql` relative to `cmd/migrate/main.go`.
- No changes to `sqlc.yaml` needed for this story — sqlc reads migrations from `migrations/` which already includes new files automatically.

### References

- Foundation migration pattern: [_bmad-output/implementation-artifacts/fourdogs-central/dev-story-1-2-database-schema-migrations.md](_bmad-output/implementation-artifacts/fourdogs-central/dev-story-1-2-database-schema-migrations.md)
- sqlc schema path: `sqlc.yaml` → `schema: "migrations/"`
- Architecture: [_bmad-output/lens-work/initiatives/fourdogs/central/centralauthorization/architecture.md](_bmad-output/lens-work/initiatives/fourdogs/central/centralauthorization/architecture.md)
- Stories source: [_bmad-output/lens-work/initiatives/fourdogs/central/centralauthorization/stories.md](_bmad-output/lens-work/initiatives/fourdogs/central/centralauthorization/stories.md)

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
