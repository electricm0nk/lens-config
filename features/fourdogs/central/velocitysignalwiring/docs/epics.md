---
stepsCompleted: [1, 2, 3]
inputDocuments:
  - docs/fourdogs/central/velocitysignalwiring/architecture.md
---

# fourdogs-central-velocitysignalwiring - Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for fourdogs-central-velocitysignalwiring, decomposing the requirements from the Architecture document into implementable stories. (tech-change track — no PRD or UX artifact.)

## Requirements Inventory

### Functional Requirements

FR1: Add vc TEXT column to items table via migration 022
FR2: VelocityClassifier executes UPDATE SQL anchored to MAX(sale_date) - 60 days
FR3: Items with upw ≥ 2.0 classified as 'fast'; ≥ 0.5 as 'medium'; otherwise 'slow'
FR4: Items with no sales in 60-day window retain vc = NULL (not updated to 'slow')
FR5: POST /internal/velocity/refresh endpoint triggers classifier on demand
FR6: Endpoint accepts optional Bearer token auth (token from VELOCITY_REFRESH_TOKEN env var)
FR7: Daily 4am goroutine in cmd/api calls ClassifyVelocity directly
FR8: emailfetcher calls POST /internal/velocity/refresh when salesRowsImported > 0
FR9: use_vendor_catalog.ts maps row.vc → ChairSku.velocity (null/missing → 'slow')
FR10: sqlc regeneration after migration makes vc appear in GetItemsByVendorRow
FR11: EtailPet trigger stub RequestDailySalesReport() exists in emailfetcher, fires at most once/day, returns nil

### NonFunctional Requirements

NFR1: Classifier response includes duration_ms and items_classified count
NFR2: NULL anchor (empty sales table) does not error — returns 0 items_classified
NFR3: Tests must pass: go test ./internal/handler/... and npm test order-lifecycle
NFR4: Migration is reversible (022_add_vc_to_items.down.sql drops column)
NFR5: velocity.go and velocity_test.go are the canonical test surface for the classifier
NFR6: No new binary — everything runs in existing fourdogs-central and emailfetcher containers

### Additional Requirements

- sqlc regeneration required after migration 022 is applied (vc appears as pgtype.Text in GetItemsByVendorRow)
- AD-2: emailfetcher triggers classifier via HTTP outbound call (POST /internal/velocity/refresh); classifier logic stays in cmd/api
- EtailPet stub guarded by last_triggered_date (max once per calendar day)
- Party mode review: itoa → strconv.FormatInt in velocity_test.go (Magos Domina)
- Party mode review: TestClassifyVelocity_EmptySalesTable missing (Artemis)
- Party mode review: sales_anchor field in response (Artemis)
- velocity.go + velocity_test.go already partially implemented — polish + test completeness required
- migrations/022 files already exist
- config.go VelocityRefreshToken already wired
- cmd/api/main.go route + scheduler already wired

### FR Coverage Map

FR1:  Epic 1 — migration 022 vc column on items table
FR2:  Epic 1 — classifier SQL UPDATE with MAX(sale_date) anchor
FR3:  Epic 1 — threshold logic fast/medium/slow
FR4:  Epic 1 — NULL preservation for no-sales items
FR5:  Epic 1 — POST /internal/velocity/refresh handler
FR6:  Epic 1 — Bearer token auth on refresh endpoint
FR7:  Epic 1 — daily 4am goroutine in cmd/api
FR8:  Epic 3 — emailfetcher post-import HTTP trigger
FR9:  Epic 2 — use_vendor_catalog.ts vc mapping
FR10: Epic 2 — sqlc regen GetItemsByVendorRow.Vc
FR11: Epic 3 — EtailPet stub
NFR1: Epic 1 — response body duration_ms + items_classified
NFR2: Epic 1 — empty sales guard (NULL anchor → 0 items, no error)
NFR3: Epic 1+2 — handler tests + frontend tests pass
NFR4: Epic 1 — down migration (022_add_vc_to_items.down.sql)
NFR5: Epic 1 — velocity_test.go completeness
NFR6: Epic 3 — no new binary constraint

## Epic List

### Epic 1: DB Foundation + Classifier API
The velocity classifier is fully operational: the vc column exists in production, the UPDATE SQL is correct and hardened, the on-demand API endpoint works with auth, and the daily scheduler fires. The HOT signal has a real data source it can read from.
**FRs covered:** FR1, FR2, FR3, FR4, FR5, FR6, FR7, NFR1, NFR2, NFR3, NFR4, NFR5

### Epic 2: Frontend Signal Wiring
The ordering worksheet's HOT badge fires from real velocity data. The frontend reads `row.vc` from the API response and maps it to the velocity tier correctly.
**FRs covered:** FR9, FR10, NFR3

### Epic 3: emailfetcher Integration + EtailPet Stub
Every completed sales import cycle automatically triggers velocity reclassification via HTTP. The EtailPet daily report request stub is wired and guarded against spam.
**FRs covered:** FR8, FR11, NFR6

---

## Epic 1: DB Foundation + Classifier API

The velocity classifier is fully operational: the vc column exists in production, the UPDATE SQL is correct and hardened, the on-demand API endpoint works with auth, and the daily scheduler fires. The HOT signal has a real data source it can read from.

### Story 1.1: Apply Migration and Verify vc Column

As a developer,
I want migration 022 applied to the production database,
So that the items table has the vc TEXT column that the classifier can write to.

**Acceptance Criteria:**

**Given** the migration file `migrations/022_add_vc_to_items.up.sql` exists with `ALTER TABLE items ADD COLUMN IF NOT EXISTS vc TEXT`
**When** the migration is applied against the production database
**Then** `items.vc` column exists as nullable TEXT
**And** all existing rows have `vc = NULL` (no default value assigned)
**And** the down migration `022_add_vc_to_items.down.sql` can cleanly reverse it

**Given** sqlc generate is run after the migration
**When** `GetItemsByVendorRow` is inspected
**Then** a `Vc pgtype.Text` field with JSON tag `"vc"` is present in the generated struct
**And** `go build ./...` completes with no errors

### Story 1.2: Harden Classifier Logic and Tests

As a developer,
I want the velocity classifier to be complete, correct, and fully tested,
So that it reliably classifies items and handles all edge cases before being connected to production triggers.

**Acceptance Criteria:**

**Given** `internal/handler/velocity.go` exists with `ClassifyVelocity()` and `VelocityRefresh()`
**When** `ClassifyVelocity()` is called against a DB with sales data
**Then** items with upw ≥ 2.0 are SET to `'fast'`
**And** items with 0.5 ≤ upw < 2.0 are SET to `'medium'`
**And** items with upw < 0.5 are SET to `'slow'`
**And** items not appearing in the 60-day window retain their existing vc value (NULL stays NULL)

**Given** `internal/handler/velocity_test.go` is reviewed against party mode findings
**When** the hand-rolled `itoa()` helper is present
**Then** it is replaced with `strconv.FormatInt(n, 10)` (Magos Domina finding)

**Given** the sales table is empty (MAX(sale_date) returns NULL)
**When** `ClassifyVelocity()` is called
**Then** it returns `0, nil` — zero items classified, no error (NFR2)
**And** the test `TestClassifyVelocity_EmptySalesTable` covers this case (Artemis finding)

**Given** `VelocityRefresh()` is called
**When** it returns a response body
**Then** the JSON includes `status`, `items_classified`, `duration_ms`, and `sales_anchor` fields
**And** `sales_anchor` is the ISO date string of `MAX(sale_date)`, or `null` if sales table is empty (Artemis finding)

**Given** `go test ./internal/handler/...` is run
**When** all tests execute
**Then** all tests pass with no failures

### Story 1.3: Wire Route, Auth, and Daily Scheduler

As a developer,
I want the velocity refresh endpoint and daily scheduler fully wired in cmd/api,
So that the classifier can be triggered on-demand (with optional auth) and automatically at 4am daily.

**Acceptance Criteria:**

**Given** `cmd/api/main.go` has `r.Post("/internal/velocity/refresh", handler.VelocityRefresh(pool, cfg.VelocityRefreshToken))`
**When** a POST request is made to `/internal/velocity/refresh` with no Authorization header
**And** `VELOCITY_REFRESH_TOKEN` env var is empty
**Then** the request is accepted and classifier runs (open endpoint)

**Given** `VELOCITY_REFRESH_TOKEN` is set to a non-empty value
**When** a POST request is made without `Authorization: Bearer <token>`
**Then** the response is `401 Unauthorized`

**When** a POST request is made with the correct `Authorization: Bearer <token>`
**Then** the classifier runs and returns `{"status":"ok","items_classified":N,"duration_ms":M,"sales_anchor":"YYYY-MM-DD"}`

**Given** `go scheduleDailyVelocityRefresh(ctx, pool)` is running as a goroutine in main
**When** the local clock passes 04:00
**Then** `ClassifyVelocity()` is called directly (not via HTTP)
**And** the result is logged (items classified or error)
**And** the goroutine respects `ctx.Done()` on shutdown

**Given** `internal/config/config.go` has `VelocityRefreshToken string`
**When** `Load()` is called
**Then** the token is populated from `os.Getenv("VELOCITY_REFRESH_TOKEN")`

**Given** `go build ./...` is run
**When** compilation completes
**Then** there are no errors

---

## Epic 2: Frontend Signal Wiring

The ordering worksheet's HOT badge fires from real velocity data. The frontend reads `row.vc` from the API response and maps it to the velocity tier correctly.

### Story 2.1: Map vc Field to Velocity Tier in useVendorCatalog

As an operator using the ordering worksheet,
I want the HOT badge to reflect real sales velocity data,
So that fast-moving items are accurately flagged and I can prioritize reorders accordingly.

**Acceptance Criteria:**

**Given** `src/hooks/use_vendor_catalog.ts` currently has `velocity: 'medium'` hardcoded at line ~77
**When** the hook is updated to read `row.vc` from the API response
**Then** the mapping logic is:
```typescript
const vcRaw = typeof row.vc === 'string' ? row.vc.trim().toLowerCase() : ''
const velocity: ChairSku['velocity'] =
  vcRaw === 'fast' ? 'fast' : vcRaw === 'medium' ? 'medium' : 'slow'
```
**And** `null`, `undefined`, empty string, or any unrecognised value maps to `'slow'`
**And** the velocity constant replaces the hardcoded `'medium'` in the returned object

**Given** `GetItemsByVendorRow` now includes `Vc pgtype.Text` after sqlc regen (Story 1.1)
**When** the API serializes the row to JSON
**Then** the response includes `"vc": "fast"` / `"vc": "medium"` / `"vc": "slow"` / `"vc": null`
**And** the frontend reads this field correctly

**Given** an item has `vc = 'fast'` in the DB
**When** it appears in the ordering worksheet
**Then** `sku.velocity === 'fast'` evaluates to true
**And** the HOT badge renders

**Given** an item has `vc = NULL` in the DB
**When** it appears in the ordering worksheet
**Then** `sku.velocity === 'slow'` (no HOT badge)

**Given** `npm test -- --run src/__tests__/order-lifecycle.test.tsx` is run
**When** all tests execute
**Then** all 13 tests pass with no failures

---

## Epic 3: emailfetcher Integration + EtailPet Stub

Every completed sales import cycle automatically triggers velocity reclassification via HTTP. The EtailPet daily report request stub is wired and guarded against spam.

### Story 3.1: emailfetcher Post-Import Velocity Trigger

As a system operator,
I want velocity classifications to update automatically after each sales import,
So that the HOT badge reflects data from the most recently imported sales without manual intervention.

**Acceptance Criteria:**

**Given** `internal/emailfetcher/runcycle.go` runs a cycle that imports sales rows
**When** `salesRowsImported > 0` after the `SalesImporter` completes
**Then** emailfetcher makes a POST request to `http://localhost:8080/internal/velocity/refresh` (or configured URL)
**And** the request includes `Authorization: Bearer <VELOCITY_REFRESH_TOKEN>` if the token is configured
**And** a non-2xx response is logged as a warning but does not abort the emailfetcher cycle

**When** `salesRowsImported == 0`
**Then** no HTTP call is made (no unnecessary classifier runs)

**Given** the central API is temporarily unavailable
**When** the HTTP call fails or times out (timeout ≤ 5 seconds)
**Then** emailfetcher logs the error and continues normally (NFR6 — no new binary, no crash)

**Given** `go build ./...` is run on the emailfetcher
**When** compilation completes
**Then** there are no errors

### Story 3.2: EtailPet Daily Report Request Stub

As a developer,
I want the EtailPet outbound trigger stub wired into the emailfetcher main loop,
So that the intended future automation path is declared explicitly and can be implemented when vendor API credentials are available.

**Acceptance Criteria:**

**Given** `internal/emailfetcher/etailpet_trigger.go` exists
**When** `RequestDailySalesReport(ctx, cfg)` is called
**Then** it logs `"etailpet_trigger: stub — daily report request not yet implemented"` via slog
**And** it returns `nil` (no error, no side effects)

**Given** the emailfetcher main loop calls `RequestDailySalesReport`
**When** it fires
**Then** it is guarded by a `last_triggered_date` check so it fires at most once per calendar day
**And** the guard prevents triggering on every 5-minute poll cycle

**Given** `EtailPetEnabled` config field exists in emailfetcher config
**When** it is `false` (default)
**Then** `RequestDailySalesReport` is not called at all (opt-in)

**Given** `go build ./...` is run on the emailfetcher
**When** compilation completes
**Then** there are no errors

