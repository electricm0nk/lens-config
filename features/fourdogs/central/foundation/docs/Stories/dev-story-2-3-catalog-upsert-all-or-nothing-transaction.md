# Story 2.3: Catalog Upsert — All-or-Nothing Transaction

Status: ready-for-dev

## Story

As Todd,
I want the ingestion script to write all validated records to Postgres in a single transaction,
so that the catalog is either fully updated or left entirely unchanged — never partially written.

## Acceptance Criteria

1. All N records written within a single `pgx.Tx`
2. Uses `sqlc UpsertItem` (INSERT ON CONFLICT (i) DO UPDATE) for each record
3. Prints `[3/4] Upserting to Postgres... N upserted (X new, Y updated, 0 skipped)`
4. Prints `[4/4] Done. Catalog is current as of {ISO8601}` on success; exits 0
5. Any single row DB error → rollback; exit 1 with "Aborting. No records written."
6. Same CSV run twice → identical output, no errors (idempotent)

## Tasks / Subtasks

- [ ] Task 0: Verify Story 2.2 complete — `ComputedItem` slice is available from pipeline

- [ ] Task 1: Write failing tests for upsert (AC: 1, 5, 6) — RED phase
  - [ ] Create `internal/ingest/upsert_test.go` (integration tests, `//go:build integration`):
    - Test: `UpsertAll(ctx, tx, items)` — all items written, tx committed, `items` count matches DB count
    - Test: forced DB error on row N → rollback → DB count remains 0 (no partial writes)
    - Test: run UpsertAll twice with same items → second run succeeds, DB count unchanged (idempotent)
  - [ ] Run `go test -tags integration ./internal/ingest/...` — RED (no implementation)

- [ ] Task 2: Implement `internal/ingest/upsert.go` (AC: 1, 2, 5) — GREEN phase
  - [ ] Add pgx/v5 dependency: `go get github.com/jackc/pgx/v5`
  - [ ] `UpsertAll(ctx context.Context, pool *pgxpool.Pool, items []ComputedItem) (newCount, updatedCount int, err error)`:
    - Begin `pool.Begin(ctx)` → `pgx.Tx`
    - Initialize `db.New(tx)` sqlc queries
    - For each item: call `q.UpsertItem(ctx, db.UpsertItemParams{...})`
    - On any error: `tx.Rollback(ctx)`; return error
    - All items written: `tx.Commit(ctx)`
    - Return new/updated counts (if tracking needed; otherwise simplify to just total count)
  - [ ] Note: INSERT ON CONFLICT DO UPDATE — `new` count is not easily derivable without row-level result inspection; consider simplifying to "N upserted" without breaking down new vs updated
  - [ ] Run `go test -tags integration ./internal/ingest/...` — GREEN
  - [ ] Commit: `feat(ingest): add all-or-nothing upsert transaction`

- [ ] Task 3: Wire upsert into CLI (AC: 3, 4, 5)
  - [ ] `cmd/ingest/main.go`: after signals, establish pgxpool connection, call `ingest.UpsertAll`
  - [ ] Print `[3/4] Upserting to Postgres... N upserted`
  - [ ] On upsert error: print "Aborting. No records written." + exit 1
  - [ ] On success: print `[4/4] Done. Catalog is current as of {time.Now().UTC().Format(time.RFC3339)}`; exit 0
  - [ ] Commit: `feat(ingest): wire upsert stage into CLI; complete pipeline stages`

- [ ] Task 4: Idempotency test (AC: 6)
  - [ ] Run ingestion pipeline end-to-end with test CSV twice
  - [ ] Verify second run: same output, same DB row count, no errors
  - [ ] Verify: `SELECT COUNT(*) FROM items` returns same value after both runs

- [ ] Task 5: Update sprint status
  - [ ] Set `2-3-catalog-upsert-all-or-nothing-transaction: done`

## Dev Notes

- `pgxpool.Pool` is constructed once in `cmd/ingest/main.go` using `DATABASE_URL` env var: `pgxpool.New(ctx, os.Getenv("DATABASE_URL"))`
- NFR2: single `pgx.Tx` — ALL item writes in one transaction; rollback on any error
- NFR3: idempotency — INSERT ON CONFLICT DO UPDATE ensures re-running is safe
- The sqlc `UpsertItem` uses `ON CONFLICT (i) DO UPDATE` — `i` is the primary key; identical second run updates rows in-place with identical values = no-op functionally
- FR20: no service restart required — the API reads from the same Postgres table; after commit, the next `/v1/items` call reflects updated data
- For counting new vs updated: simplest approach is to not count separately (both cases are handled by upsert); just report total N upserted
  - More complex: use `INSERT ... ON CONFLICT ... RETURNING xmax` — `xmax = 0` means INSERT (new); `xmax != 0` means UPDATE; adds complexity, optional
- `pgx.Tx` vs `pgxpool.Tx`: use `pool.BeginTx(ctx, pgx.TxOptions{})` for clean semantics
- Deferred rollback pattern:
  ```go
  defer func() {
      if err != nil {
          tx.Rollback(ctx)
      }
  }()
  ```

### Project Structure Notes

- `internal/ingest/upsert.go` — transaction logic
- `internal/db/` — sqlc generated code (UpsertItem params)
- `cmd/ingest/main.go` — pool construction + pipeline orchestration

### References

- [Source: phases/techplan/architecture.md — Ingestion, ARCH5 pgxpool]
- [Source: phases/devproposal/epics.md — Story 2.3 ACs, NFR2, NFR3, FR20]
- [Source: phases/businessplan/prd.md — FR5 (incremental refresh), FR13]

## Dev Agent Record

### Agent Model Used

GitHub Copilot / Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
