# Story 1.1: Apply Migration and Verify vc Column

Status: ready-for-dev

## Story

As a developer,
I want migration 022 applied to the production database and the sqlc-generated `GetItemsByVendorRow` struct updated to include the `vc` field,
so that the velocity classifier has a column to write to and the API response automatically carries velocity classification data.

## Context

**Initiative:** fourdogs-central-velocitysignalwiring (tech-change)
**Epic:** 1 — DB Foundation + Classifier API
**Dependency:** None — this is the first story; Stories 1.2, 2.1 depend on this being done first.
**Repo:** fourdogs-central (Go), branch `fourdogs-central-velocitysignalwiring`

### Why this matters

The HOT badge fires `sku.velocity === 'fast'`. Today `velocity` is hardcoded `'medium'` in the frontend hook. All three wiring stories (1.2, 1.3, 2.1) depend on `vc TEXT` existing in `items` and appearing in the JSON response from `GET /v1/items?vendor_id=X`. Without this story, the classifier has nowhere to write and the frontend has no field to read.

## Acceptance Criteria

**Given** migration file `migrations/022_add_vc_to_items.up.sql` contains:
```sql
ALTER TABLE items ADD COLUMN IF NOT EXISTS vc TEXT;
```
**When** the migration is applied against the production database  
**Then** `items.vc` column exists as nullable TEXT  
**And** all existing rows have `vc = NULL` (no default)  
**And** the down migration `022_add_vc_to_items.down.sql` can cleanly reverse it:
```sql
ALTER TABLE items DROP COLUMN IF EXISTS vc;
```

**Given** `i.vc` is added to the explicit SELECT list in `internal/db/queries/items.sql` (GetItemsByVendor query)  
**When** `sqlc generate` is run  
**Then** `GetItemsByVendorRow` struct in `internal/db/items.sql.go` includes:
```go
Vc pgtype.Text `json:"vc"`
```
**And** `go build ./...` completes with no errors

## Tasks / Subtasks

- [ ] Apply migration 022 to production DB (AC: #1)
  - [ ] Run migration against production: use existing migration runner in `migrations/migrations.go`
  - [ ] Verify: `psql $DB_URL -c "\d items"` shows `vc | text` column
  - [ ] Verify: `SELECT COUNT(*) FROM items WHERE vc IS NOT NULL` returns 0 (all NULL initially)
- [ ] Add `i.vc` to the GetItemsByVendor SELECT query (AC: #2)
  - [ ] Edit `internal/db/queries/items.sql` — GetItemsByVendor query (line ~64–86)
  - [ ] Add `i.vc,` after `i.t` in the SELECT list (at the end, before FROM)
  - [ ] ⚠️ CAUTION: This query has an explicit column list — NOT `SELECT i.*`. Adding `i.vc` to the query is REQUIRED for sqlc to include it in the struct. The migration alone is not sufficient.
- [ ] Run sqlc generate (AC: #2)
  - [ ] `sqlc generate` from repo root
  - [ ] Verify `GetItemsByVendorRow` in `internal/db/items.sql.go` now has `Vc pgtype.Text` field
  - [ ] Check scan order in the generated `rows.Scan(...)` call matches the SELECT column order
- [ ] Verify build (AC: #2)
  - [ ] `go build ./...` with zero errors
  - [ ] `go test ./internal/handler ./internal/auth` — all tests pass (no new test changes needed)

## Dev Notes

### Pre-existing Implementation State

Everything in this story is setup work. The migration files already exist on the initiative branch:

```
migrations/022_add_vc_to_items.up.sql    ← EXISTS: ALTER TABLE items ADD COLUMN IF NOT EXISTS vc TEXT;
migrations/022_add_vc_to_items.down.sql  ← EXISTS: ALTER TABLE items DROP COLUMN IF EXISTS vc;
```

**No migration file changes needed.** The task is only to *apply* the migration to production, then update the SQL query and regenerate.

### Current GetItemsByVendorRow struct (pre-migration)

```go
// internal/db/items.sql.go — CURRENT (before this story)
type GetItemsByVendorRow struct {
    I  string        `json:"i"`
    U  pgtype.Text   `json:"u"`
    N  pgtype.Text   `json:"n"`
    B  pgtype.Text   `json:"b"`
    Q  pgtype.Int4   `json:"q"`
    V  pgtype.Float8 `json:"v"`
    Va pgtype.Float8 `json:"va"`
    C  pgtype.Int4   `json:"c"`
    K  pgtype.Float8 `json:"k"`
    P  pgtype.Float8 `json:"p"`
    F  pgtype.Int2   `json:"f"`
    O  pgtype.Int4   `json:"o"`
    S  pgtype.Int2   `json:"s"`
    X  pgtype.Int2   `json:"x"`
    Y  pgtype.Int2   `json:"y"`
    Sp pgtype.Text   `json:"sp"`
    T  pgtype.Text   `json:"t"`
    // Vc pgtype.Text — MISSING, added after migration + sqlc regen
}
```

### Required SQL query change

File: `internal/db/queries/items.sql`
Query: `-- name: GetItemsByVendor :many` (line ~58)

Add `i.vc,` to the explicit SELECT list — append after `i.t` (line ~82), before the FROM clause:

```sql
-- Before the FROM clause, add:
    i.vc
```

Full column list after change ends with:
```sql
    i.sp,
    i.t,
    i.vc
FROM vendors v
...
```

### sqlc regen behavior

After `sqlc generate`, the generated struct will automatically add:
```go
Vc pgtype.Text `json:"vc"`
```
And the scan call in the generated function body will add the field to its `rows.Scan(...)` call matching the SELECT column order.

⚠️ **Do not manually edit `internal/db/items.sql.go`** — it is entirely generated by sqlc. Any hand edits will be overwritten on next regen.

### Production DB connection

The production DB URL is available as the `DATABASE_URL` env var in the running pod. For direct psql access during verification, the URL is available via:
```bash
kubectl get secret -n fourdogs-central-dev fourdogs-central-secrets -o jsonpath='{.data.DATABASE_URL}' | base64 -d
```
or use the credentials available to the developer from the team secrets store.

### GetItemsByVendorAdapter — separate query, same change needed?

**Check:** `internal/db/queries/items.sql` also contains `-- name: GetItemsByVendorAdapter :many` (line ~14).  
**Assessment:** If the adapter variant also has an explicit SELECT list and is serialized to the frontend, `i.vc` should also be added there for consistency. Review during implementation and decide. If the adapter path is not surfaced in the frontend, it can be left for a follow-up.

### Migration runner

Migrations are applied automatically at startup via `migrations/migrations.go`. The migration may have already been applied to dev environments during development. Check with:
```sql
SELECT vc FROM items LIMIT 1;
-- If no error → migration already applied
-- If "column does not exist" → needs to be applied
```

### Project structure
```
fourdogs-central/
├── migrations/
│   ├── 022_add_vc_to_items.up.sql    ← EXISTS (apply to prod)
│   ├── 022_add_vc_to_items.down.sql  ← EXISTS (reversible)
│   └── migrations.go                 (migration runner)
├── internal/db/
│   ├── queries/items.sql             ← MODIFY: add i.vc to GetItemsByVendor SELECT
│   ├── items.sql.go                  ← GENERATED: will get Vc field after sqlc regen
│   └── models.go                     (may need review — Item struct)
└── internal/handler/
    └── velocity.go                   ← Story 1.2 scope (do not touch yet)
```

### References

- Architecture AD-1: `docs/fourdogs/central/velocitysignalwiring/architecture.md#AD-1`
- Architecture AD-3 (SQL pattern): `docs/fourdogs/central/velocitysignalwiring/architecture.md#AD-3`
- Epics FR1, FR10: `docs/fourdogs/central/velocitysignalwiring/epics.md#Story-1.1`
- sqlc docs: [sqlc.dev](https://sqlc.dev) — explicit SELECT columns map 1:1 to struct fields

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
- `internal/db/queries/items.sql` (MODIFY — add i.vc to GetItemsByVendor SELECT)
- `internal/db/items.sql.go` (GENERATED — do not edit directly; regenerated by sqlc)
- `migrations/022_add_vc_to_items.up.sql` (READ-ONLY — apply to prod, no edits)
