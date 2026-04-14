---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
lastStep: 8
status: complete
completedAt: '2026-04-14'
inputDocuments:
  - docs/fourdogs/central/architecture.md
  - docs/fourdogs/central/central-ui/architecture.md
  - docs/fourdogs/central/prd.md
  - docs/fourdogs/central/foundation/Stories/dev-story-2-2-velocity-and-ordering-signal-computation.md
workflowType: architecture
project_name: fourdogs-central-velocitysignalwiring
user_name: Todd
date: '2026-04-14'
initiative: fourdogs-central-velocitysignalwiring
---

# Architecture Decision Document — fourdogs-central-velocitysignalwiring

**Author:** Todd  
**Date:** 2026-04-14  
**Track:** tech-change  
**Scope:** Velocity signal wiring — DB-driven HOT classification + EtailPet trigger stub

---

## Project Context Analysis

### Problem Statement

The fourdogs-central ordering worksheet surfaces a `HOT` signal on catalog items to indicate fast-moving inventory. This signal is evaluated against `sku.velocity === 'fast'` in the frontend. However, `useVendorCatalog.ts` unconditionally assigns `velocity: 'medium'` to all items, bypassing the `v` field already computed by the ingest pipeline and stored in `items.v`. The result: the HOT signal never fires from real data.

A corrected velocity signal requires three things working end-to-end:
1. The DB `items` table exposes a classifed velocity tier (`fast`/`medium`/`slow`/`null`) — not a raw float
2. The classifier is recomputed automatically after each sales import cycle
3. The frontend reads the DB-backed tier instead of the hardcoded default

### Data Foundation (confirmed against production DB)

| Metric | Value |
|---|---|
| Sales rows in DB | 28,067 |
| Date range | 2025-01-02 – 2026-03-31 |
| Items with any sales (60-day window) | 1,531 |
| Items with no sales in 60 days | 3,969 |
| Total items | ~5,500 |

**Threshold sensitivity against real data (60-day window, cutoff 2026-01-14):**

| Fast threshold | Fast items | Medium band (≥0.5 upw) |
|---|---|---|
| ≥ 1.5 upw | 125 | 220 |
| ≥ 2.0 upw | 86 | 259 |
| ≥ 3.0 upw | 46 | 154 |

**Selected thresholds:**
- `fast` ≥ **2.0 units/week** (86 items — visible enough to be useful, exclusive enough to mean something)
- `medium` ≥ **0.5 and < 2.0 upw** (259 items)
- `slow` < 0.5 upw, or no sales in 60-day window
- `NULL` — item not in `items` table (should not occur in practice)

---

## Architecture Decisions

### AD-1: Storage — Column on `items`, not a separate table

**Decision:** Add `vc TEXT DEFAULT NULL` column to the existing `items` table.

**Rationale:** A separate `item_signals` table introduces a JOIN on every catalog query and operational surface area for a classification value that changes at most once per sales import cycle. The existing `GetItemsByVendor` and `GetItemsByVendorAdapter` queries use `SELECT i.*` / explicit column lists already serialized as JSON by sqlc. Adding `vc` to `items` makes it appear in the API response automatically after sqlc regeneration — no handler changes, no new endpoint. The audit trail for *when* an item changed class is not a requirement at this time; if it becomes one, a separate history table can be added without disrupting this design.

**Rejected:** Separate `item_signals` table (over-engineered for current requirements).

### AD-2: Classifier Trigger — emailfetcher calls central API endpoint, not a CronJob

**Decision:** The emailfetcher, upon detecting `salesRowsImported > 0` in a cycle, calls `POST /internal/velocity/refresh` on the central API. The central API executes the classifier UPDATE SQL. There is also a daily 4am goroutine in the central API that calls the classifier directly for time-based freshness.

**Rationale:** Sales reports arrive via email at unpredictable times (daily, but not at a fixed clock time). A wall-clock CronJob would either lag behind imports (runs at midnight, email arrives at 3pm) or recompute unnecessarily on cycles where no new sales landed. The emailfetcher already polls every 5 minutes and knows exactly how many sales rows were imported in each cycle. Triggering via HTTP call to `POST /internal/velocity/refresh` gives event-driven freshness with zero additional infrastructure and keeps the classifier logic in the central API where it has direct DB access.

**Process boundary:** The classifier logic lives in `cmd/api` (`internal/handler/velocity.go`). The emailfetcher makes an outbound HTTP call — it does not execute classifier logic in-process. This separation means the classifier can also be triggered on-demand (by the 4am scheduler, or manually) independently of the emailfetcher cycle.

**Consequence:** Velocity is not recomputed on cycles with no sales imports. This is correct — stale velocity from yesterday is more accurate than triggering on no new data.

### AD-3: Velocity window — 60 days of *available* data, not 60 calendar days

**Decision:** The classifier anchors to `MAX(sale_date) FROM sales`, then looks back 60 days from that anchor — not from `CURRENT_DATE`.

**Rationale:** Sales data is imported in batches (currently: operator downloads CSV from EtailPet and forwards to the Gmail mailbox). The most recent import may be days behind today. `CURRENT_DATE - 60` would silently shrink the effective window every day data is not imported. Anchoring to `MAX(sale_date)` ensures the classifier always evaluates the most recent 60 days of *actual data*, regardless of import cadence. When the EtailPet API trigger is wired (see AD-5), the anchor will naturally converge to near-today.

**Core SQL:**
```sql
WITH anchor AS (
  SELECT MAX(sale_date) AS max_date FROM sales
),
v60 AS (
  SELECT s.system_id,
         SUM(s.quantity) / 60.0 * 7.0 AS upw
  FROM sales s, anchor a
  WHERE s.sale_date >= a.max_date - INTERVAL '60 days'
  GROUP BY s.system_id
)
UPDATE items
SET vc = CASE
  WHEN v.upw >= 2.0 THEN 'fast'
  WHEN v.upw >= 0.5 THEN 'medium'
  ELSE 'slow'
END
FROM v60 v
WHERE items.i = v.system_id;
-- Items not appearing in v60 (no sales in window) keep vc = NULL → UI treats as slow
```

### AD-4: Frontend — Read `row.vc`, fall back to `'slow'` when null

**Decision:** `normalizeItem` in `use_vendor_catalog.ts` maps `row.vc` to `ChairSku.velocity`, with null/missing falling back to `'slow'`.

```typescript
// Before: velocity: 'medium'
// After:
const vcRaw = typeof row.vc === 'string' ? row.vc.trim().toLowerCase() : ''
const velocity: ChairSku['velocity'] =
  vcRaw === 'fast' ? 'fast' :
  vcRaw === 'medium' ? 'medium' :
  'slow'
```

**Rationale:** `null` in DB means "no sales in 60-day window." Treating that as `'slow'` is semantically correct — items with no recent sales movement should not be marked HOT. The fallback to `'slow'` rather than `'medium'` matches the actual signal semantics.

### AD-5: EtailPet trigger stub — inside emailfetcher, not a new binary

**Decision:** Add `internal/emailfetcher/etailpet_trigger.go` as a stub that will eventually call the EtailPet POS API to request the daily sales report email. The stub is wired into the main loop but returns immediately.

**Rationale:** The eventual trigger belongs in the same process as the thing that consumes the resulting email. Placing it in the emailfetcher creates a single coherent unit: *request data → wait for email → import → classify*. A separate binary or CronJob would introduce coordination complexity with no benefit. The stub makes the intended wiring explicit without blocking delivery of the classifier.

**Stub behavior:**
```go
// RequestDailySalesReport is called once per day (guarded by last-trigger timestamp)
// to ask EtailPet to send the sales report email to the monitored mailbox.
// When implemented, this enables fully automated daily velocity refresh.
// TODO: implement EtailPet API call (endpoint and auth TBD with vendor).
func RequestDailySalesReport(ctx context.Context, cfg *Config) error {
    slog.InfoContext(ctx, "etailpet_trigger: stub — daily report request not yet implemented")
    return nil
}
```

The stub is guarded by a `last_triggered_date` check so it fires at most once per calendar day (no EtailPet spam on 5-minute poll cycles).

---

## Component Map

```
emailfetcher (existing container)
├── cmd/emailfetcher/main.go          existing — unchanged
├── internal/emailfetcher/
│   ├── runcycle.go                   MODIFY — call trigger stub + post-cycle classifier hook
│   ├── config.go                     MODIFY — add EtailPetEnabled bool config field
│   ├── etailpet_trigger.go           NEW — stub for outbound POS API report request
│   └── velocity/
│       ├── classifier.go             NEW — Run(ctx, pool) executes UPDATE SQL
│       └── classifier_test.go        NEW — unit tests against test DB or mock
├── migrations/
│   └── 022_add_vc_to_items.up.sql   NEW — ALTER TABLE items ADD COLUMN vc TEXT
│   └── 022_add_vc_to_items.down.sql NEW — ALTER TABLE items DROP COLUMN vc
└── internal/db/queries/items.sql     VERIFY — vc appears in SELECT i.* (no change needed)

fourdogs-central-ui
└── src/hooks/use_vendor_catalog.ts   MODIFY — line 77: map row.vc → velocity tier
```

**sqlc regeneration required** after migration 022 is applied. `GetItemsByVendorRow.Vc` will appear as `pgtype.Text` with JSON tag `"vc"`.

---

## Data Flow (end-to-end, post-implementation)

```
[Manual today]   Operator downloads CSV → forwards to Gmail mailbox
[Future stub]    EtailPet API → sends sales report email daily (once wired)
                    ↓
emailfetcher polls Gmail every 5m
  → SalesImporter.Import() upserts rows into sales table
  → salesRowsImported > 0 → VelocityClassifier.Run()
      → anchor = MAX(sale_date) FROM sales
      → UPDATE items SET vc = 'fast'/'medium'/'slow' WHERE sales in last 60d
      → items with no sales → vc remains NULL (API returns null → UI shows 'slow')
                    ↓
GET /v1/items?vendor_id=X
  → GetItemsByVendorRow includes vc field
  → normalizeItem maps vc → ChairSku.velocity
  → OrderDetail: sku.velocity === 'fast' → HOT badge fires from real data
```

---

## Non-Goals (explicitly out of scope)

- Historical velocity trend tracking (no `item_signals` table this iteration)
- Per-vendor velocity segmentation (single store, single sales stream)
- Configurable thresholds via UI (thresholds are compile-time constants for now)
- Ingest CLI (`cmd/ingest`) velocity computation — that path uses CSV `v`/`va` fields directly and is unaffected

---

## Risk Register

| Risk | Likelihood | Mitigation |
|---|---|---|
| `sales.system_id` doesn't match `items.i` for all items | Low | Confirmed in live data query — system_id present; items join works |
| vc stays NULL if no sales email arrives for >60d | Medium | UI treats NULL as 'slow' — correct, honest reflection of data state |
| EtailPet trigger API credentials/endpoint unknown | High | Stub makes no calls — no risk until implemented |
| sqlc regeneration diverges between dev and prod | Low | CI gate on sqlc codegen already present (story 1-3) |
