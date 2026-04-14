# Story 1.4: Config tests with testdata fixtures

**Initiative:** terminus-inference-provider-routing
**Epic:** 1 — Operator can configure routing policy in YAML
**Status:** review

## Story

As a **developer**,
I want `config_test.go` to cover all `LoadProfiles` paths using YAML fixture files run with the race detector,
So that config validation behaviour is regression-tested and concurrent-access safe.

## Acceptance Criteria

**Given** `internal/routing/testdata/` contains fixture YAML files for:
- valid config
- unsupported schema version
- missing required field (`price_per_1k_tokens`)
- missing env var (api_key_env set but env var unset)
- negative budget
- undefined provider reference in profiles

**When** `go test -race ./internal/routing/...` is run
**Then** all table-driven cases in `config_test.go` pass with zero race detector findings
**And** each fixture file represents exactly one scenario (one file per case)
**And** no fixture file contains a real API key value — test env vars are set inline within the test using `t.Setenv`

## Technical Notes

- Target repo: `terminus-inference-gateway`
- Package: `package routing` (white-box testing)
- Fixture directory: `internal/routing/testdata/`
- Use `t.Setenv` to set/unset env vars per test case — no global state mutation
- Table-driven test struct: `{ name string; fixture string; setenv map[string]string; wantErr string }`
- Do not use real credentials in any fixture or test

## Dependencies

- Story 1.3 (all LoadProfiles paths implemented)
