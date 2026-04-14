# Story 1.4: Config tests with testdata fixtures

Status: ready-for-dev

## Story

As a **developer**,
I want `config_test.go` to cover all `LoadProfiles` paths using YAML fixture files run with the race detector,
So that config validation behaviour is regression-tested and concurrent-access safe.

## Acceptance Criteria

1. **Given** `internal/routing/testdata/` contains fixture YAML files for:
   - valid config
   - unsupported schema version
   - missing required field
   - missing env var
   - negative budget
   - undefined provider reference  
   **When** `go test -race ./internal/routing/...` is run  
   **Then** all table-driven cases in `config_test.go` pass with zero race detector findings

2. **And** each fixture file represents exactly one scenario (one file per case)

3. **And** no fixture file contains a real API key value — test env vars are set inline within the test

## Tasks / Subtasks

- [ ] Create `internal/routing/testdata/` directory (AC: 1)
- [ ] Create one YAML fixture file per scenario (AC: 1, 2):
  - [ ] `testdata/valid-config.yaml` — schema_version 1, 2 providers, 2 profiles
  - [ ] `testdata/unsupported-schema-version.yaml` — schema_version: 2
  - [ ] `testdata/missing-required-field.yaml` — provider missing `price_per_1k_tokens`
  - [ ] `testdata/missing-env-var.yaml` — api_key_env set to a var that won't be in test env
  - [ ] `testdata/negative-budget.yaml` — daily_budget_usd: -1.0
  - [ ] `testdata/undefined-provider-reference.yaml` — profile references "nonexistent"
- [ ] Implement `config_test.go` with table-driven tests (AC: 1, 3)
  - [ ] Each case: `name`, `fixtureFile`, `envSetup` (map[string]string), `wantErr bool`, `errContains string`
  - [ ] Set env vars inline using `t.Setenv(k, v)` — no external env state (AC: 3)
  - [ ] Valid config case: assert `LoadProfiles` returns non-nil `Router` and nil error
  - [ ] Error cases: assert `err != nil` and error message contains expected substring
- [ ] Run `go test -race ./internal/routing/...` and verify all tests pass (AC: 1)

## Dev Notes

- Test package should be `package routing` (white-box — same package as implementation) for access to unexported types
- `t.Setenv(key, value)` automatically restores original env var state after test — use this instead of `os.Setenv` to avoid test pollution
- For `missing-env-var.yaml`: set `api_key_env` to a deliberately unique name like `ROUTING_TEST_MISSING_KEY_XYZ123` and do NOT set it in the test's envSetup — this guarantees it won't be accidentally set in CI
- For `valid-config.yaml`: set `api_key_env` to a test var like `ROUTING_TEST_API_KEY` and use `envSetup: {"ROUTING_TEST_API_KEY": "test-key-value"}` in the test case
- Race detector flag: the `-race` flag exercises concurrent correctness — even for config parsing (which runs once), include it as the CI gate requirement
- Fixture files must use the same canonical schema as the architecture doc — not variations or shorthand

### Project Structure Notes

- Files: `internal/routing/config_test.go`, `internal/routing/testdata/` (6 YAML files)
- `testdata/` is a standard Go convention for test fixtures — `go test` automatically makes it available via relative paths

### References

- [Source: docs/terminus/inference/provider-routing/architecture.md#Additional Requirements] — "testdata/ subdirectory allowed for config_test.go YAML fixtures"; `go test -race ./internal/routing/...` CI gate
- [Source: docs/terminus/inference/provider-routing/epics.md#Story 1.4] — All AC scenarios
- [Source: docs/terminus/inference/provider-routing/epics.md#Story 1.2] — ValidConfig and error cases to cover

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
