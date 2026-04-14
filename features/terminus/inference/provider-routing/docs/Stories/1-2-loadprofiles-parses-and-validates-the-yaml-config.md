# Story 1.2: LoadProfiles parses and validates the YAML config

Status: ready-for-dev

## Story

As an **operator**,
I want `LoadProfiles` to parse a YAML routing config and return a validation error if anything is wrong,
So that a misconfigured gateway fails loudly at startup rather than silently misbehaving.

## Acceptance Criteria

1. **Given** `ROUTING_CONFIG_PATH` points to a valid YAML file matching the canonical schema with `schema_version: 1`  
   **When** `LoadProfiles(os.Getenv("ROUTING_CONFIG_PATH"))` is called  
   **Then** it returns a `(Router, nil)` with all providers and profiles loaded into unexported runtime state  
   **And** an `slog.Info` log line is emitted confirming load success, containing only the config file path (no content, no keys)

2. **Given** `ROUTING_CONFIG_PATH` is empty or unset  
   **When** `LoadProfiles("")` is called  
   **Then** it returns `(nil, error)` with message `"routing: ROUTING_CONFIG_PATH environment variable is not set"`

3. **Given** the YAML file declares `schema_version: 2`  
   **When** `LoadProfiles` is called  
   **Then** it returns `(nil, error)` naming the unsupported version and the supported value (`1`)

4. **Given** a required field (e.g. `price_per_1k_tokens`) is missing from a provider entry  
   **When** `LoadProfiles` is called  
   **Then** it returns `(nil, error)` naming the exact field and provider name in the message

5. **Given** `daily_budget_usd` is set to a negative value for any provider  
   **When** `LoadProfiles` is called  
   **Then** it returns `(nil, error)`; negative budget values are rejected

6. **Given** `profiles` references a provider name not declared in the `providers` list  
   **When** `LoadProfiles` is called  
   **Then** it returns `(nil, error)` naming the undefined provider reference

## Tasks / Subtasks

- [ ] Define YAML-unmarshal structs in `config.go` (AC: 1)
  - [ ] `RoutingConfig` with `SchemaVersion int`, `Providers []ProviderConfig`, `Profiles map[string]ProfileConfig`
  - [ ] `ProviderConfig` with `Name string`, `APIKeyEnv string`, `DailyBudgetUSD float64`, `PricePerKTokens float64`
  - [ ] `ProfileConfig` with `Providers []string` (priority-ordered)
- [ ] Implement `LoadProfiles(configPath string) (Router, error)` in `config.go` (AC: 1–6)
  - [ ] Empty path check → return `"routing: ROUTING_CONFIG_PATH environment variable is not set"` (AC: 2)
  - [ ] Read file and unmarshal YAML (AC: 1)
  - [ ] Validate `schema_version` == `SupportedSchemaVersion`; reject others with version in error (AC: 3)
  - [ ] Validate required fields per provider; name field + provider in error message (AC: 4)
  - [ ] Reject `daily_budget_usd < 0` (AC: 5)
  - [ ] Validate all profile provider references exist in `providers` list (AC: 6)
  - [ ] Emit `slog.Info` on success with config path only (AC: 1)
- [ ] Verify error messages follow `fmt.Errorf("routing: %s: %w", ...)` convention

## Dev Notes

- `LoadProfiles` signature: `func LoadProfiles(configPath string) (Router, error)` — exported in `config.go`
- Returns a concrete `*router` struct that implements the `Router` interface — the struct is unexported
- All validation must be completed before returning; do not partially init and return on first error — collect all errors or use fail-fast (fail-fast is acceptable per architecture)
- Canonical YAML schema (from architecture):
  ```yaml
  schema_version: 1
  providers:
    - name: openai
      api_key_env: OPENAI_API_KEY
      daily_budget_usd: 5.00
      price_per_1k_tokens: 0.002
    - name: ollama
      api_key_env: ""
      daily_budget_usd: 0
      price_per_1k_tokens: 0.0
  profiles:
    interactive:
      providers: [openai, ollama]
    batch:
      providers: [ollama, openai]
  ```
- `slog.Info` on success must include ONLY the config path — never log provider names, key env var names, or budget values at this stage
- Error message for schema version: `"routing: config: unsupported schema_version %d; supported: %d"`
- Error message for missing field: `"routing: config: provider %q missing required field %q"`
- Error message for negative budget: `"routing: config: provider %q daily_budget_usd must be >= 0"`
- Error message for undefined profile reference: `"routing: config: profile %q references unknown provider %q"`
- API key resolution happens in Story 1.3 — this story only validates structural config

### Project Structure Notes

- File: `internal/routing/config.go`
- `LoadProfiles` is the entry point used by `main.go` (Story 5.1) — keep signature stable
- Do NOT read `os.Getenv` inside `LoadProfiles`; the caller (`main.go`) passes the resolved path — this keeps the function testable without env var mocking

### References

- [Source: docs/terminus/inference/provider-routing/prd.md#FR14–FR17] — YAML schema, validation, startup log
- [Source: docs/terminus/inference/provider-routing/architecture.md#Additional Requirements] — `SupportedSchemaVersion = 1`, canonical schema, error format
- [Source: docs/terminus/inference/provider-routing/epics.md#Story 1.2] — All AC scenarios

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
