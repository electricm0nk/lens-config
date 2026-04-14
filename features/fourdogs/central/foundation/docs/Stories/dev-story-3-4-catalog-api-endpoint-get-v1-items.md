# Story 3.4: Catalog API Endpoint — GET /v1/items

Status: ready-for-dev

## Story

As Betsy,
I want to retrieve the item catalog from the API,
so that I can browse and manage my ordering session using live data instead of a static file.

## Acceptance Criteria

1. `GET /v1/items` returns a bare JSON array — no envelope object, no pagination (UX7)
2. Each item contains all 17 exact field names: `i`, `u`, `n`, `b`, `q`, `v`, `va`, `c`, `k`, `p`, `f`, `o`, `s`, `x`, `y`, `sp`, `t` (NFR1, FR7)
3. Only items from active brands are returned — a brand is active if it has at least one item where `y=1` (FR8)
4. Empty catalog returns `[]` (empty JSON array), not `null`
5. P99 response time under 2 seconds on ~900 records (NFR4)
6. Unauthenticated request → HTTP 401 (enforced by Story 3.3 middleware; no change needed here)

## Tasks / Subtasks

- [ ] Task 0: Verify Story 3.3 complete — session middleware gating `/v1/*` routes

- [ ] Task 1: Verify sqlc query exists for `GetItemsByBrand` (AC: 2, 3)
  - [ ] Check `sql/queries/items.sql` — query must return rows with exact column aliases `i`, `u`, `n`, `b`, `q`, `v`, `va`, `c`, `k`, `p`, `f`, `o`, `s`, `x`, `y`, `sp`, `t`
  - [ ] Verify `sqlc.yaml` uses `json_tags_case_style: none` so generated struct fields have json tags matching column names (NOT TitleCase)
  - [ ] If `sqlc.yaml` does not yet specify `json_tags_case_style`, add it:
    ```yaml
    sql:
      - schema: sql/schema
        queries: sql/queries
        engine: postgresql
        gen:
          go:
            package: db
            out: internal/db
            json_tags_case_style: none  # ← critical: preserves i, u, n, b, q, etc.
    ```
  - [ ] Run `sqlc generate` to regenerate; verify `internal/db/items.sql.go` has json tags `json:"i"`, `json:"u"`, etc.
  - [ ] If query does not have active-brands filter: add `WHERE b IN (SELECT brand FROM items WHERE y = true)` or equivalent
  - [ ] Run `go test ./internal/db/...` — GREEN

- [ ] Task 2: Write failing handler tests (AC: 1, 2, 3, 4) — RED phase
  - [ ] Create `internal/handler/items_test.go`:
    - Test: returns HTTP 200 with `Content-Type: application/json`
    - Test: response body is a JSON array (first char `[`, last char `]`)
    - Test: each item has all 17 required fields (spot-check via `json.Unmarshal` into `map[string]interface{}`)
    - Test: inactive brand items excluded (mock queries to return only active-brand items)
    - Test: empty result set → response is `[]` not `null` (critical: nil slice encodes as null in Go — use `make([]db.GetItemsByBrandRow, 0)`)
  - [ ] Use mock `db.Querier` interface for isolation
  - [ ] Run `go test ./internal/handler/...` — RED

- [ ] Task 3: Implement `GET /v1/items` handler (AC: 1, 2, 3, 4)
  - [ ] Create/update `internal/handler/items.go`:
    ```go
    func GetItems(queries db.Querier) http.HandlerFunc {
        return func(w http.ResponseWriter, r *http.Request) {
            items, err := queries.GetItemsByBrand(r.Context())
            if err != nil {
                http.Error(w, "internal error", http.StatusInternalServerError)
                return
            }
            if items == nil {
                items = make([]db.GetItemsByBrandRow, 0) // prevent null JSON
            }
            w.Header().Set("Content-Type", "application/json")
            json.NewEncoder(w).Encode(items)
        }
    }
    ```
  - [ ] NOT: `{"items": [...]}` — bare array only (UX7)
  - [ ] NOT: `null` on empty result — always `[]`
  - [ ] Run `go test ./internal/handler/...` — GREEN

- [ ] Task 4: Wire handler into protected router group (AC: 1, 6)
  - [ ] Add to protected group in `cmd/api/main.go`:
    ```go
    r.Get("/v1/items", handler.GetItems(queries))
    ```
  - [ ] Verify session middleware applies (from Story 3.3 router group)
  - [ ] Run `go test ./...` — GREEN
  - [ ] Commit: `feat(api): implement GET /v1/items catalog endpoint`

- [ ] Task 5: Performance validation (AC: 5 — NFR4, sub-2s P99)
  - [ ] With k3s deployment: run `hey -n 200 -c 10 https://fourdogs.terminus.local/v1/items` (with valid session cookie)
  - [ ] Target: P99 < 2000ms on ~900 records
  - [ ] If slow: add `EXPLAIN ANALYZE` on `GetItemsByBrand`; check that `b` column in items table is indexed
  - [ ] Record P99 result in completion notes

- [ ] Task 6: Update sprint status
  - [ ] Set `3-4-catalog-api-endpoint-get-v1-items: done`

## Dev Notes

- **Critical:** `json_tags_case_style: none` in `sqlc.yaml` — without this, sqlc generates TitleCase json tags like `json:"I"` instead of `json:"i"`, which breaks prototype parity (NFR1)
- **Nil slice vs empty slice:** In Go, `var s []T` serializes to `null`; `s := make([]T, 0)` serializes to `[]`. The AC explicitly requires `[]` on empty catalog. Check your DB query result handling
- **No envelope:** `{"data": [...]}` or `{"items": [...]}` is WRONG. Bare `[{...}, ...]`
- **Active brands filter (FR8):** Only return items where the item's brand has at least one item with `y=1`. This filter prevents items from deprecated/disabled brands from appearing. The SQL should be:
  ```sql
  -- name: GetItemsByBrand :many
  SELECT i, u, n, b, q, v, va, c, k, p, f, o, s, x, y, sp, t
  FROM items
  WHERE b IN (
    SELECT DISTINCT b FROM items WHERE y = true
  )
  ORDER BY b, n;
  ```
- **17 exact field names:** `i` (SKU/id), `u` (UPC), `n` (name), `b` (brand), `q` (qty), `v` (vendor), `va` (vendor alt), `c` (category), `k` (pack), `p` (price), `f` (flag), `o` (order qty), `s` (status), `x` (extra), `y` (active), `sp` (special), `t` (type) — match exactly what prototype `loader.html` expects
- Story 3.3 middleware handles auth — this handler has NO auth logic of its own

### Source Tree

```
internal/
  handler/
    items.go            ← this story
    items_test.go       ← this story
  db/
    items.sql.go        ← generated by sqlc (verify json tags)
sql/
  queries/
    items.sql           ← GetItemsByBrand query
sqlc.yaml               ← verify json_tags_case_style: none
cmd/
  api/
    main.go             ← wire /v1/items into protected group
```

### References

- [Source: phases/devproposal/epics.md — Story 3.4 ACs, FR7, FR8, NFR1, NFR4, UX7]
- [Source: phases/businessplan/prd.md — FR7 (exact field names), FR8 (active brands filter)]
- [Source: phases/techplan/architecture.md — API response format, no envelope]

## Dev Agent Record

### Agent Model Used

GitHub Copilot / Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
