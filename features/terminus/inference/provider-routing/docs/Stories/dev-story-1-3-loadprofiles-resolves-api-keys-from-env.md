# Story 1.3: LoadProfiles resolves API keys from environment variables

**Initiative:** terminus-inference-provider-routing
**Epic:** 1 — Operator can configure routing policy in YAML
**Status:** review

## Story

As an **operator**,
I want provider API keys to be referenced by env var name in config and resolved at startup,
So that no plaintext credentials ever appear in the YAML file or leak through the system.

## Acceptance Criteria

**Given** a provider entry has `api_key_env: OPENAI_API_KEY` and the env var is set to a non-empty value
**When** `LoadProfiles` is called
**Then** the resolved key is stored in an unexported runtime field on the provider's internal entry
**And** the resolved key value is never present in any exported type, log output, error message, or `HealthStatus`

**Given** `api_key_env: OPENAI_API_KEY` and the env var is NOT set (empty or missing)
**When** `LoadProfiles` is called
**Then** it returns `(nil, error)` with a message that names the provider and the missing env var name
**And** the error message does NOT contain any credential value

**Given** `api_key_env: ""` (empty string — e.g. a local unauthenticated provider)
**When** `LoadProfiles` is called
**Then** no env var lookup is attempted and the provider loads successfully with an empty key

## Technical Notes

- Target repo: `terminus-inference-gateway`
- Implement within `LoadProfiles` in `config.go`
- Use an unexported struct field (e.g. `resolvedAPIKey string`) on the internal provider entry
- Key resolution via `os.Getenv(provider.APIKeyEnv)` — called once at startup, result stored
- Keys must never appear in: exported types, `HealthStatus`, `ProviderStatus`, log records, error messages
- Slog allow-list for provider entries: only `name` and `base_url` are loggable

## Dependencies

- Story 1.2 (LoadProfiles base implementation)
