# Story 2.2: Velocity and Ordering Signal Computation

Status: ready-for-dev

## Story

As the system,
I want the ingestion pipeline to compute per-item velocity metrics and suggested order quantities,
so that each catalog record reflects current sales-derived ordering signals at the time of ingest.

## Acceptance Criteria

1. `v` (recent 13-week velocity, units/week) computed per item from sales history
2. `va` (overall ~51-week velocity) computed per item
3. `o` (suggested order quantity): 10-day demand ceiling rounded up to nearest case size `c`, floored at 0
4. `y=1` (system recommendation flag) when `o > 0`; `y=0` otherwise
5. `f=1` (date-check flag) when `c == 111`
6. Stage output: `[2/4] Computing signals... N items processed`

## Tasks / Subtasks

- [ ] Task 0: Verify Story 2.1 complete — CSV parsing returns `RawRecord` slice

- [ ] Task 1: Write failing tests for signal computation (AC: 1–5) — RED phase
  - [ ] Create `internal/ingest/signals_test.go`:
    - Test: `ComputeSignals(records)` → `v` = (sum of 13 recent weeks) / 13
    - Test: `ComputeSignals(records)` → `va` = (sum of ~51 weeks) / 51
    - Test: `o` = ceil(v * 10/7) rounded up to nearest multiple of `c`, min 0
    - Test: `o > 0` → `y = 1`; `o == 0` → `y = 0`
    - Test: `c == 111` → `f = 1`; any other `c` → `f = 0`
    - Test: zero-velocity item → `o = 0`, `y = 0`
    - Test: case size `c = 0` (edge case) → handled without divide-by-zero
  - [ ] Run `go test ./internal/ingest/...` — confirm RED (signals not implemented)

- [ ] Task 2: Define `ComputedItem` type
  - [ ] Add `internal/ingest/types.go` with:
    ```go
    type ComputedItem struct {
        I  string
        U  string
        N  string
        B  string
        Q  int
        V  float64
        Va float64
        C  int
        K  float64
        P  float64
        F  int16   // date-check flag (c == 111 → 1)
        O  int     // computed suggested order qty
        S  int16   // special-request flag (set in Story 2.4)
        X  int16   // OOS flag (set in Story 2.4)
        Y  int16   // system recommendation (o > 0 → 1)
        Sp string  // species (set in Story 2.4)
        T  string  // product type (set in Story 2.4)
    }
    ```

- [ ] Task 3: Implement `internal/ingest/signals.go` (AC: 1–5) — GREEN phase
  - [ ] Parse `RawRecord` fields from string → numeric types (use `strconv.ParseFloat`, `strconv.Atoi`)
  - [ ] Compute `v` from EtailPet sales history columns (13 most recent weekly sales columns)
  - [ ] Compute `va` from EtailPet sales history columns (all available weekly columns up to 51)
  - [ ] Compute `o`:
    ```go
    func suggestedQty(v float64, c int) int {
        if v <= 0 || c <= 0 {
            return 0
        }
        demandFor10Days := v * (10.0 / 7.0)
        return int(math.Ceil(demandFor10Days / float64(c))) * c
    }
    ```
  - [ ] Set `y = 1` if `o > 0`
  - [ ] Set `f = 1` if `c == 111`
  - [ ] `ComputeSignals(records []RawRecord) ([]ComputedItem, error)`:
    - Converts each `RawRecord` to `ComputedItem` with all signals set
    - Returns error on type conversion failure (bad numeric data)
  - [ ] Run `go test ./internal/ingest/...` — all tests pass (GREEN)
  - [ ] Commit: `feat(ingest): add velocity and ordering signal computation`

- [ ] Task 4: Wire signals into CLI (AC: 6)
  - [ ] `cmd/ingest/main.go`: after parsing, call `ingest.ComputeSignals(records)`
  - [ ] Print `[2/4] Computing signals... N items processed`
  - [ ] Pass `ComputedItem` slice to upsert stage (stub for now — Story 2.3)
  - [ ] Commit: `feat(ingest): wire ComputeSignals into CLI pipeline`

- [ ] Task 5: Update sprint status
  - [ ] Set `2-2-velocity-and-ordering-signal-computation: done`

## Dev Notes

- EtailPet "sales history" columns: the exact column naming depends on the real export — need sample export from Todd
  - Typical format: weekly sales for recent 13 weeks in columns like `Week 1`, `Week 2`, ... `Week 13` (most recent to oldest)
  - For 51-week (`va`), use all available weekly columns; if < 51, divide by actual count
- Formula for `o`: 10-day demand window (10/7 × v) rounded up to nearest case size
  - `o = ceil(v × 10/7 / c) × c` — roundup to nearest multiple of case size
  - If `c = 0` or `v = 0`, `o = 0`; no divide-by-zero
- `f=1` when `c == 111` — this is a sentinel value in EtailPet meaning "date-check product" (not an actual case size of 111)
- `import "math"` for `math.Ceil`
- S and X flags are left at 0 here — set in Story 2.4 classification pass

### Project Structure Notes

- `internal/ingest/fieldmap.go` — column names (from Story 2.1)
- `internal/ingest/parse.go` — returns `[]RawRecord` (from Story 2.1)
- `internal/ingest/signals.go` — converts `RawRecord` → `ComputedItem` with numeric signals
- `internal/ingest/types.go` — shared types

### References

- [Source: phases/techplan/architecture.md — Ingestion Data Flow]
- [Source: phases/devproposal/epics.md — Story 2.2 ACs, FR3, FR4, FR15]
- [Source: phases/businessplan/prd.md — FR14 (velocity from sales history), FR15 (order qty)]

## Dev Agent Record

### Agent Model Used

GitHub Copilot / Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
