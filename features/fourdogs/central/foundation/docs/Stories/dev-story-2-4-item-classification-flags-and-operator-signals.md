# Story 2.4: Item Classification, Flags, and Operator Signals

Status: ready-for-dev

## Story

As the system,
I want ingestion to classify each item by species and product type and apply all operator-facing flags,
so that the API can filter and surface correct ordering signals without any post-processing.

## Acceptance Criteria

1. `sp` (species) derived from category path: `dog | cat | small_animal | fish | bird | reptile | general`
2. `t` (product type) derived from category path: `food | treats | toys | other`
3. `x=1` (OOS flag) for items from known OOS distributor sources; their `o` set to 0 regardless of velocity
4. `s=1` (special-request flag) for designated items; appear in catalog regardless of `y` value
5. All classification logic in `internal/ingest/ingest.go` — no mapping in `cmd/ingest`
6. All 4 flag types (`f`, `x`, `s`, `y`) correct on representative EtailPet test fixture
7. Unit tests in `internal/ingest/ingest_test.go` cover classification edge cases; unknown category → `general` / `other`

## Tasks / Subtasks

- [ ] Task 0: Verify Story 2.2/2.3 complete — `ComputedItem` in pipeline

- [ ] Task 1: Write failing tests for classification (AC: 1, 2, 7) — RED phase
  - [ ] Add to `internal/ingest/ingest_test.go` (or create `internal/ingest/classify_test.go`):
    - Test: `classifySpecies("Dog/Food/Dry")` → `"dog"`
    - Test: `classifySpecies("Cat/Treats")` → `"cat"`
    - Test: `classifySpecies("Small Animal/Bedding")` → `"small_animal"`
    - Test: `classifySpecies("Aquatics/Tank")` → `"fish"`
    - Test: `classifySpecies("Bird/Seed")` → `"bird"`
    - Test: `classifySpecies("Reptile/Habitat")` → `"reptile"`
    - Test: `classifySpecies("")` → `"general"` (unknown path fallback)
    - Test: `classifySpecies("UnknownCategory")` → `"general"`
    - Test: `classifyType("Dog/Food/Dry")` → `"food"`
    - Test: `classifyType("Dog/Treats")` → `"treats"`
    - Test: `classifyType("Dog/Toys")` → `"toys"`
    - Test: `classifyType("")` → `"other"`
    - Test: `classifyType("Accessories/Collars")` → `"other"`
  - [ ] Run `go test ./internal/ingest/...` — RED

- [ ] Task 2: Implement `internal/ingest/classify.go` (AC: 1, 2) — GREEN phase
  - [ ] `classifySpecies(categoryPath string) string`:
    ```go
    func classifySpecies(path string) string {
        lower := strings.ToLower(path)
        switch {
        case strings.HasPrefix(lower, "dog"):
            return "dog"
        case strings.HasPrefix(lower, "cat"):
            return "cat"
        case strings.Contains(lower, "small animal"):
            return "small_animal"
        case strings.Contains(lower, "aquatic") || strings.Contains(lower, "fish"):
            return "fish"
        case strings.HasPrefix(lower, "bird"):
            return "bird"
        case strings.HasPrefix(lower, "reptile"):
            return "reptile"
        default:
            return "general"
        }
    }
    ```
  - [ ] `classifyType(categoryPath string) string`:
    ```go
    func classifyType(path string) string {
        lower := strings.ToLower(path)
        switch {
        case strings.Contains(lower, "food") || strings.Contains(lower, "nutrition"):
            return "food"
        case strings.Contains(lower, "treat"):
            return "treats"
        case strings.Contains(lower, "toy"):
            return "toys"
        default:
            return "other"
        }
    }
    ```
  - [ ] Note: exact matching strategy depends on real EtailPet category path format — adjust after seeing real data
  - [ ] Run `go test ./internal/ingest/...` — GREEN
  - [ ] Commit: `feat(ingest): add species and product type classification`

- [ ] Task 3: Implement OOS and special-request flag logic (AC: 3, 4)
  - [ ] Determine OOS distributor identification method from EtailPet data (field or source identifier)
  - [ ] `applyFlags(item *ComputedItem, raw RawRecord)`:
    - Set `item.X = 1` if item's distributor source is in the OOS sources list; also set `item.O = 0`
    - Set `item.S = 1` if item has special-request marker in EtailPet data (determine marker from sample export)
    - Set `item.Sp` and `item.T` from `classifySpecies` and `classifyType`
  - [ ] Add OOS sources list as a package-level variable (configurable without code change ideally, but package var acceptable for MVP)
  - [ ] Commit: `feat(ingest): apply OOS, special-request, and classification flags`

- [ ] Task 4: Integrate into pipeline as classification pass (AC: 5)
  - [ ] In `internal/ingest/` (or `ingest.go`): after `ComputeSignals`, call `applyFlags` on each item
  - [ ] The `ComputeSignals` function should call classification internally or a new pipeline stage function should apply classification
  - [ ] Ensure no classification logic leaks into `cmd/ingest`
  - [ ] Commit: `feat(ingest): integrate classification pass into ingest pipeline`

- [ ] Task 5: Validate against test fixture (AC: 6)
  - [ ] Add representative EtailPet test fixture to `internal/ingest/testdata/`
  - [ ] Run `go test ./internal/ingest/...` — all 4 flag types (`f`, `x`, `s`, `y`) set correctly
  - [ ] Run end-to-end CLI pipeline: `go run ./cmd/ingest testdata/representative_fixture.csv --dry-run`

- [ ] Task 6: Update sprint status
  - [ ] Set `2-4-item-classification-flags-and-operator-signals: done`
  - [ ] Set `epic-2: done`
  - [ ] Set `epic-3: in-progress`

## Dev Notes

- Category path format in EtailPet: likely slash-separated hierarchy (e.g., "Dog/Food/Dry Dog Food") — verify with real export before implementing matching
- OOS source identification: may be a distributor field in the CSV (e.g., "Distributor: OUT_OF_STOCK_VENDOR") or a product-level flag — clarify with Todd based on actual export
- Special-request items: may be flagged via a custom EtailPet field or a maintained list — clarify implementation approach with Todd
- `x=1` items: `o` is forced to 0 (override the signal computation result) — must happen AFTER signal computation, not before
- `s=1` items: included in API response even when `y=0` — this is enforced in the `GetItemsByBrand` SQL query (Story 3.4 review may need to adjust query to include `s=1` items)
- Classification tests use simple string matching — adjust after seeing real EtailPet category paths
- All flag logic: `internal/ingest` package, not `cmd/ingest`

### Project Structure Notes

- `internal/ingest/classify.go` — species and type classification
- `internal/ingest/flags.go` (or merged into classify.go) — x, s flag application
- Pipeline order: ParseCSV → ComputeSignals → ApplyClassificationAndFlags → UpsertAll

### References

- [Source: phases/techplan/architecture.md — Ingestion, classification design]
- [Source: phases/devproposal/epics.md — Story 2.4 ACs, FR16, FR17, FR18, FR19]
- [Source: phases/businessplan/prd.md — FR16 (OOS flag), FR17 (special-request), FR18 (date-check), FR19 (classification)]

## Dev Agent Record

### Agent Model Used

GitHub Copilot / Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
