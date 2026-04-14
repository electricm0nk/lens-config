# Story 1.2: LoadProfiles parses and validates the YAML config

**Initiative:** terminus-inference-provider-routing
**Epic:** 1 — Operator can configure routing policy in YAML
**Status:** review

## Story

As an **operator**,
I want `LoadProfiles` to parse a YAML routing config and return a validation error if anything is wrong,
So that a misconfigured gateway fails loudly at startup rather than silently misbehaving.

## Acceptance Criteria

**Given** `ROUTING_CONFIG_PATH` points to a valid YAML file matching the canonical schema with `schema_version: 1`
**When** `LoadProfiles(os.Getenv("ROUTING_CONFIG_PATH"))` is called
**Then** it returns a `(Router, nil)` with all providers and profiles loaded into unexported runtime state
**And** an `slog.Info` log line is emitted confirming load success, containing only the config file path (no content, no keys)

**Given** `ROUTING_CONFIG_PATH` is empty or unset
**When** `LoadProfiles("")` is called
**Then** it returns `(nil, error)` with message `"routing: ROUTING_CONFIG_PATH environment variable is not set"`

**Given** the YAML file declares `schema_version: 2`
**When** `LoadProfiles` is called
**Then** it returns `(nil, error)` naming the unsupported version and the supported value (`1`)

**Given** a required field (e.g. `price_per_1k_tokens`) is missing from a provider entry
**When** `LoadProfiles` is called
**Then** it returns `(nil, error)` naming the exact field and provider name in the message

**Given** `daily_budget_usd` is set to a negative value for any provider
**When** `LoadProfiles` is called
**Then** it returns `(nil, error)`; negative budget values are rejected

**Given** `profiles` references a provider name not declared in the `providers` list
**When** `LoadProfiles` is called
**Then** it returns `(nil, error)` naming the undefined provider reference

## Technical Notes

- Target repo: `terminus-inference-gateway`
- Implement `LoadProfiles(configPath string) (Router, error)` in `config.go`
- Canonical YAML schema:
  ```yaml
  schema_version: 1
  providers:
    - name: ollama
      base_url: http://ollama.inference.svc.cluster.local:11434
      api_key_env: ""          # empty = no auth required
      daily_budget_usd: 0      # 0 = unlimited
      price_per_1k_tokens: 0   # required even if unlimited
    - name: openai
      base_url: https://api.openai.com/v1
      api_key_env: OPENAI_API_KEY
      daily_budget_usd: 5.00
      price_per_1k_tokens: 0.002
  profiles:
    - name: interactive
      provider_chain: [openai, ollama]
    - name: batch
      provider_chain: [ollama, openai]
  ```
- Error wrapping convention: `fmt.Errorf("routing: load config: %w", err)`
- `slog.Info` on success: log the config path only, never log provider entries or key values

## Dependencies

- Story 1.1 (package scaffold must exist)
