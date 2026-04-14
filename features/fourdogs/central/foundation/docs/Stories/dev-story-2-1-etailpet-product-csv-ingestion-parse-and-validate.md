# Story 2.1: EtailPet Product CSV Ingestion — Parse and Validate

Status: ready-for-dev

## Story

As Todd (operator),
I want the ingestion script to parse and validate an EtailPet Product Detail Report CSV,
so that malformed or missing-field records are caught before any DB writes occur.

## Acceptance Criteria

1. No file path argument → print usage and exit 1
2. Valid EtailPet CSV: parses + maps EtailPet column headers to 17 `items` schema fields; prints `[1/4] Parsing CSV... N records read`
3. `--dry-run` flag: validate without writing to DB
4. CSV rows with missing required fields: print `Row N: missing field (field_name)` per error; exit 1 with "Aborting. No records written. Fix CSV and re-run." — no DB writes

## Tasks / Subtasks

- [ ] Task 0: Verify Epic 1 complete — `make migrate-local` runs cleanly

- [ ] Task 1: Write failing tests for CSV parser (AC: 1, 2, 4) — RED phase
  - [ ] Create `internal/ingest/ingest_test.go`:
    - Test: `ParseCSV("")` → error "file path required"
    - Test: `ParseCSV("testdata/valid.csv")` → returns N records, no error
    - Test: `ParseCSV("testdata/missing_fields.csv")` → returns ValidationError with row numbers and field names
    - Test: all 17 schema fields map correctly from EtailPet column headers
  - [ ] Create `internal/ingest/testdata/valid.csv` — minimal EtailPet CSV fixture (3-5 rows, all fields)
  - [ ] Create `internal/ingest/testdata/missing_fields.csv` — fixture with 1 row missing a required field
  - [ ] Run `go test ./internal/ingest/...` — confirm fail (RED)

- [ ] Task 2: Implement CSV column header mapping (AC: 2) — GREEN phase
  - [ ] Determine EtailPet column header names (from sample export or product documentation)
  - [ ] Create `internal/ingest/fieldmap.go` with a `columnMap` mapping EtailPet header → `items` field:
    ```go
    var columnMap = map[string]string{
        "Product ID":     "i",    // unique item identifier (EtailPet product ID)
        "UPC":            "u",    // distributor UPC
        "Product Name":   "n",    // display name
        "Brand":          "b",    // brand name
        "QOH":            "q",    // quantity on hand
        // ... add all 17 fields
        "Category":       "",     // used for sp/t classification — not stored directly
    }
    ```
  - [ ] Note: exact header names must match the real EtailPet Product Detail Report — verify against an actual export

- [ ] Task 3: Implement `internal/ingest/parse.go` (AC: 1, 2, 3, 4)
  - [ ] `ParseCSV(path string) ([]RawRecord, []ValidationError, error)`:
    - Opens file; returns error if path empty or file not found
    - Reads header row; fails if required column not present
    - Maps each data row to `RawRecord` (map[string]string)
    - Validates required fields present for each row; accumulates `ValidationError{Row int, Field string}`
    - Returns records + errors (not early-exit; collect all errors)
  - [ ] `RawRecord` type: `map[string]string` keyed by items schema field names
  - [ ] Run `go test ./internal/ingest/...` — must pass (GREEN)

- [ ] Task 4: Implement `cmd/ingest/main.go` CLI (AC: 1, 2, 3, 4)
  - [ ] Parse flags: `csvPath string`, `dryRun bool`
  - [ ] No args/path → print usage + exit 1
  - [ ] Call `ingest.ParseCSV(csvPath)` → collect errors
  - [ ] Print `[1/4] Parsing CSV... N records read`
  - [ ] If validation errors > 0: print each `Row N: missing field (field_name)`; exit 1 with abort message
  - [ ] If `--dry-run`: print count + "Dry run complete. No records written." + exit 0
  - [ ] If no errors + not dry-run: continue to Stage 2 (signals) — pass records to next function (stub for now, implemented in Story 2.2)
  - [ ] Commit: `feat(ingest): add CSV parsing and CLI with dry-run flag`

- [ ] Task 5: End-to-end test with real fixture (AC: 2, 3, 4)
  - [ ] Run `go run ./cmd/ingest testdata/valid.csv --dry-run` — expect clean output
  - [ ] Run `go run ./cmd/ingest testdata/missing_fields.csv` — expect row error + abort
  - [ ] Run `go run ./cmd/ingest` (no args) — expect usage + exit 1

- [ ] Task 6: Update sprint status
  - [ ] Set `2-1-etailpet-product-csv-ingestion-parse-and-validate: done`

## Dev Notes

- The EtailPet column → items field mapping is CRITICAL — exact column names depend on the real export format
- Ask Todd for a sample EtailPet Product Detail Report CSV before implementing fieldmap.go
- `RawRecord` at this stage is `map[string]string` — type conversion (string → int/float64) happens in signal computation (Story 2.2)
- Required fields for the 17 items columns: all except computed fields (v, va, o, y, f which are computed in Story 2.2; sp, t which are derived in Story 2.4)
- CSV reader: use `encoding/csv` stdlib — no external dependency needed
- UX1: stage output format `[1/4] Parsing CSV... N records read` — must match exactly
- UX2: error format `Row N: missing field (field_name)` — must match exactly; N is 1-indexed (header is row 0)
- UX3: `--dry-run` exits 0 after validation — no DB connection needed in dry-run mode at all

### Project Structure Notes

- `internal/ingest/` — all business logic; `cmd/ingest/` is thin CLI entry point only
- Test fixtures in `internal/ingest/testdata/` — checked into repo
- `fieldmap.go` is the only place EtailPet-specific knowledge lives — a single source of truth for column mapping

### References

- [Source: phases/techplan/architecture.md — Ingestion section]
- [Source: phases/devproposal/epics.md — Story 2.1 ACs, UX1, UX2, UX3]
- [Source: phases/businessplan/prd.md — FR13 (CLI ingestion), FR2 (17 fields)]

## Dev Agent Record

### Agent Model Used

GitHub Copilot / Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
