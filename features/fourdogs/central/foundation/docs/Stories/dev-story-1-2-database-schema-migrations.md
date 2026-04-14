# Story 1.2: Database Schema Migrations

Status: ready-for-dev

## Story

As the system,
I want versioned schema migrations that create the `items` and `sessions` tables,
so that the data model is reproducible, reversible, and runnable as a k8s init container.

## Acceptance Criteria

1. `migrations/001_create_items.up.sql` creates `items` table with all 17 catalog fields: `i` TEXT PK, `u` TEXT, `n` TEXT, `b` TEXT, `q` INT, `v` FLOAT8, `va` FLOAT8, `c` INT, `k` FLOAT8, `p` FLOAT8, `f` SMALLINT, `o` INT, `s` SMALLINT, `x` SMALLINT, `y` SMALLINT, `sp` TEXT, `t` TEXT
2. `idx_items_distributor_upc` index created on `u`
3. `migrations/002_create_sessions.up.sql` creates `sessions` table: `google_sub` TEXT PK, `access_token` TEXT, `refresh_token` TEXT, `expiry` TIMESTAMPTZ, `updated_at` TIMESTAMPTZ
4. Both `.down.sql` files cleanly reverse migrations with no errors
5. `cmd/migrate/main.go` uses `//go:embed migrations/*` and runs golang-migrate up/down against `DATABASE_URL`
6. Running migrations twice (idempotent check) produces no errors

## Tasks / Subtasks

- [ ] Task 0: Verify Story 1.1 complete — `go build ./...` passes on scaffold

- [ ] Task 1: Write failing tests for migrate command (AC: 5, 6) — RED phase
  - [ ] Create `cmd/migrate/main_test.go`:
    - Test: main with no `DATABASE_URL` exits non-zero
    - Test: "up" command on clean DB produces no error (integration test, requires test DB)
    - Test: running "up" twice produces no error (idempotent)
  - [ ] Mark integration tests with `//go:build integration` build tag
  - [ ] Run unit tests: `go test ./cmd/migrate/...` — fail (no implementation)

- [ ] Task 2: Create SQL migration files (AC: 1, 2, 3, 4)
  - [ ] `migrations/001_create_items.up.sql`:
    ```sql
    CREATE TABLE IF NOT EXISTS items (
        i   TEXT PRIMARY KEY,
        u   TEXT,
        n   TEXT,
        b   TEXT,
        q   INT,
        v   FLOAT8,
        va  FLOAT8,
        c   INT,
        k   FLOAT8,
        p   FLOAT8,
        f   SMALLINT,
        o   INT,
        s   SMALLINT,
        x   SMALLINT,
        y   SMALLINT,
        sp  TEXT,
        t   TEXT
    );
    CREATE INDEX IF NOT EXISTS idx_items_distributor_upc ON items (u);
    ```
  - [ ] `migrations/001_create_items.down.sql`:
    ```sql
    DROP INDEX IF EXISTS idx_items_distributor_upc;
    DROP TABLE IF EXISTS items;
    ```
  - [ ] `migrations/002_create_sessions.up.sql`:
    ```sql
    CREATE TABLE IF NOT EXISTS sessions (
        google_sub    TEXT PRIMARY KEY,
        access_token  TEXT NOT NULL,
        refresh_token TEXT,
        expiry        TIMESTAMPTZ,
        updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
    );
    ```
  - [ ] `migrations/002_create_sessions.down.sql`:
    ```sql
    DROP TABLE IF EXISTS sessions;
    ```
  - [ ] Commit: `feat(migrations): add items and sessions table migrations`

- [ ] Task 3: Implement `cmd/migrate/main.go` with go:embed (AC: 5) — GREEN phase
  - [ ] Add dependency: `github.com/golang-migrate/migrate/v4`
  - [ ] `cmd/migrate/main.go`:
    ```go
    package main

    import (
        "embed"
        "fmt"
        "log"
        "os"

        "github.com/golang-migrate/migrate/v4"
        _ "github.com/golang-migrate/migrate/v4/database/postgres"
        "github.com/golang-migrate/migrate/v4/source/iofs"
    )

    //go:embed ../../migrations/*.sql
    var migrationFS embed.FS

    func main() {
        dbURL := os.Getenv("DATABASE_URL")
        if dbURL == "" {
            log.Fatal("DATABASE_URL is required")
        }

        direction := "up"
        if len(os.Args) > 1 {
            direction = os.Args[1]
        }

        d, err := iofs.New(migrationFS, "migrations")
        if err != nil {
            log.Fatalf("failed to create migration source: %v", err)
        }

        m, err := migrate.NewWithSourceInstance("iofs", d, dbURL)
        if err != nil {
            log.Fatalf("failed to create migrator: %v", err)
        }
        defer m.Close()

        switch direction {
        case "up":
            if err := m.Up(); err != nil && err != migrate.ErrNoChange {
                log.Fatalf("migration up failed: %v", err)
            }
            fmt.Println("migrations applied successfully")
        case "down":
            if err := m.Down(); err != nil && err != migrate.ErrNoChange {
                log.Fatalf("migration down failed: %v", err)
            }
            fmt.Println("migrations rolled back successfully")
        default:
            log.Fatalf("unknown direction: %s (use 'up' or 'down')", direction)
        }
    }
    ```
  - [ ] Note: embed path is relative to the file — adjust if directory layout differs
  - [ ] Run `go build ./cmd/migrate/...` — must compile clean
  - [ ] Run integration tests against local Postgres: `go test -tags integration ./cmd/migrate/...`
  - [ ] Run `make migrate-local` twice — must succeed both times (AC: 6)
  - [ ] Commit: `feat(migrate): implement cmd/migrate with go:embed and golang-migrate`

- [ ] Task 4: Update sprint status
  - [ ] Set `1-2-database-schema-migrations: done`

## Dev Notes

- golang-migrate v4: `github.com/golang-migrate/migrate/v4`
- postgres driver: `github.com/golang-migrate/migrate/v4/database/postgres`
- iofs source lets us use `embed.FS` directly — cleaner than file URI
- The embed path `../../migrations/*.sql` is relative to `cmd/migrate/main.go` — adjust if needed
- For k8s init container: the binary must be self-contained (embedded migrations), no volume mounts
- `pgx/v5` driver with golang-migrate: use `github.com/golang-migrate/migrate/v4/database/pgx/v5` if using pgx5
- ARCH4: `google_sub` is TEXT PK on sessions — not a sequence/int; this is intentional for forward-compat
- Field names in `items` table are intentionally abbreviated (i, u, n, b, etc.) — matches prototype JSON field names exactly (NFR1)

### Project Structure Notes

- `//go:embed` import path: the directive is relative to the Go source file location
- `cmd/migrate/` has its own `main.go` — not part of `internal/` package graph
- golang-migrate `iofs.New` requires go1.16+ (available in go1.22)

### References

- [Source: phases/techplan/architecture.md — Data Model section]
- [Source: phases/devproposal/epics.md — Story 1.2 ACs]
- [Source: phases/businessplan/prd.md — FR1, FR2 (items fields), ARCH4 (google_sub PK)]

## Dev Agent Record

### Agent Model Used

GitHub Copilot / Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
