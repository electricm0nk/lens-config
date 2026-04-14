# Story 1.3: sqlc Configuration and Codegen CI Gate

Status: ready-for-dev

## Story

As a developer,
I want sqlc configured with annotated query files and generated output committed to the repo,
so that type-safe DB access is available and schema compliance is machine-enforced in CI.

## Acceptance Criteria

1. `sqlc.yaml` configured: engine=postgresql, schema=migrations/, queries=internal/db/queries/, output=internal/db/
2. `internal/db/queries/items.sql` contains `GetItemsByBrand` (all items where brand has ≥1 item with `y=1`) and `UpsertItem` (insert-or-update on PK `i`)
3. `internal/db/queries/sessions.sql` contains `GetSession`, `UpsertSession`, `DeleteSession` keyed on `google_sub`
4. Generated `internal/db/items.sql.go` and `internal/db/sessions.sql.go` committed to repo
5. `sqlc generate` produces no errors and no diff on clean working tree
6. CI `sqlc generate` diff-check step detects stale file and fails before Docker build

## Tasks / Subtasks

- [ ] Task 0: Verify Story 1.2 complete — migrations run cleanly

- [ ] Task 1: Write failing CI test for sqlc diff (AC: 6) — RED phase
  - [ ] Sketch out the CI `sqlc-check` job (implemented fully in Story 1.4)
  - [ ] For now: write a local script `scripts/check-sqlc-diff.sh`:
    ```bash
    #!/bin/bash
    sqlc generate
    git diff --exit-code internal/db/ || { echo "sqlc output is stale — run 'make generate'"; exit 1; }
    ```
  - [ ] Confirm script would fail if `internal/db/` were modified without regenerating

- [ ] Task 2: Add sqlc dependency and create `sqlc.yaml` (AC: 1)
  - [ ] `go install github.com/sqlc-dev/sqlc/cmd/sqlc@latest` (or use Makefile generate target)
  - [ ] Create `sqlc.yaml`:
    ```yaml
    version: "2"
    sql:
      - engine: "postgresql"
        queries: "internal/db/queries/"
        schema: "migrations/"
        gen:
          go:
            package: "db"
            out: "internal/db"
            emit_json_tags: true
            emit_prepared_queries: false
            emit_interface: false
            emit_exact_table_names: false
    ```
  - [ ] Commit: `chore(sqlc): add sqlc.yaml configuration`

- [ ] Task 3: Write SQL query files (AC: 2, 3)
  - [ ] `internal/db/queries/items.sql`:
    ```sql
    -- name: GetItemsByBrand :many
    SELECT i.* FROM items i
    WHERE i.b IN (
        SELECT DISTINCT b FROM items WHERE y = 1
    )
    ORDER BY i.b, i.n;

    -- name: UpsertItem :exec
    INSERT INTO items (i, u, n, b, q, v, va, c, k, p, f, o, s, x, y, sp, t)
    VALUES (@i, @u, @n, @b, @q, @v, @va, @c, @k, @p, @f, @o, @s, @x, @y, @sp, @t)
    ON CONFLICT (i) DO UPDATE SET
        u = EXCLUDED.u, n = EXCLUDED.n, b = EXCLUDED.b,
        q = EXCLUDED.q, v = EXCLUDED.v, va = EXCLUDED.va,
        c = EXCLUDED.c, k = EXCLUDED.k, p = EXCLUDED.p,
        f = EXCLUDED.f, o = EXCLUDED.o, s = EXCLUDED.s,
        x = EXCLUDED.x, y = EXCLUDED.y, sp = EXCLUDED.sp, t = EXCLUDED.t;
    ```
  - [ ] `internal/db/queries/sessions.sql`:
    ```sql
    -- name: GetSession :one
    SELECT * FROM sessions WHERE google_sub = @google_sub;

    -- name: UpsertSession :exec
    INSERT INTO sessions (google_sub, access_token, refresh_token, expiry, updated_at)
    VALUES (@google_sub, @access_token, @refresh_token, @expiry, NOW())
    ON CONFLICT (google_sub) DO UPDATE SET
        access_token = EXCLUDED.access_token,
        refresh_token = EXCLUDED.refresh_token,
        expiry = EXCLUDED.expiry,
        updated_at = NOW();

    -- name: DeleteSession :exec
    DELETE FROM sessions WHERE google_sub = @google_sub;
    ```
  - [ ] Commit: `feat(db): add sqlc query files for items and sessions`

- [ ] Task 4: Run `sqlc generate` and commit output (AC: 4, 5) — GREEN phase
  - [ ] Ensure local Postgres has migrations applied: `make migrate-local`
  - [ ] Run: `sqlc generate`
  - [ ] Verify generated files:
    - `internal/db/items.sql.go` — has `GetItemsByBrand`, `UpsertItem`
    - `internal/db/sessions.sql.go` — has `GetSession`, `UpsertSession`, `DeleteSession`
    - `internal/db/db.go` — DBTX interface
    - `internal/db/models.go` — `Item` and `Session` structs
  - [ ] Run `go build ./...` — must compile clean
  - [ ] Run `sqlc generate` again — `git diff internal/db/` must be empty (no diff = AC: 5)
  - [ ] Commit: `feat(db): add sqlc generated db package`

- [ ] Task 5: Write Go tests for generated types (AC: 4, 5)
  - [ ] Create `internal/db/db_test.go`:
    - Test: `Item` struct has fields `I`, `U`, `N`, `B` etc. (type assertion test)
    - Test: `Session` struct has `GoogleSub` field as primary key type
  - [ ] These are compile-time guarantees — test file serves as documentation
  - [ ] Run `go test ./internal/db/...`
  - [ ] Commit: `test(db): add type assertion tests for generated structs`

- [ ] Task 6: Update sprint status
  - [ ] Set `1-3-sqlc-configuration-and-codegen-ci-gate: done`

## Dev Notes

- sqlc v2 config format (not v1) — use `version: "2"`
- `@param_name` syntax in sqlc queries (pgx v5 uses named params with `@`)
- Generated `Item.I` maps to `items.i` — sqlc uses TitleCase for field names
- `GetItemsByBrand` uses a subquery to find brands with any recommended item (`y=1`) — must return ALL items from those brands (not just y=1 items), so the ordering session shows full brand catalog
- `emit_json_tags: true` is required — the handler marshals `Item` directly to JSON; json tags must match the short field names (`i`, `u`, `n`, etc.) for backwards-compat with prototype (NFR1, FR7)
- Verify sqlc generates correct json tags: `json:"i"` not `json:"I"`; check sqlc.yaml for `json_tags_case_style: camel` vs `none`

### Project Structure Notes

- `internal/db/` directory: owned by sqlc; do not hand-edit generated files
- `internal/db/queries/` directory: hand-written SQL; not generated
- Migrations in `migrations/` must be applied before sqlc can verify schema

### References

- [Source: phases/techplan/architecture.md — sqlc codegen section]
- [Source: phases/devproposal/epics.md — Story 1.3 ACs, ARCH2]
- [Source: phases/businessplan/prd.md — FR7 (exact field names), NFR1 (schema immutable)]

## Dev Agent Record

### Agent Model Used

GitHub Copilot / Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
