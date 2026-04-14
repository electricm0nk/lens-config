# Story 1.3: LoadProfiles resolves API keys from environment variables

Status: ready-for-dev

## Story

As an **operator**,
I want provider API keys to be referenced by env var name in config and resolved at startup,
So that no plaintext credentials ever appear in the YAML file or leak through the system.

## Acceptance Criteria

1. **Given** a provider entry has `api_key_env: OPENAI_API_KEY` and the env var is set to a non-empty value  
   **When** `LoadProfiles` is called  
   **Then** the resolved key is stored in an unexported runtime field on the provider's internal entry  
   **And** the resolved key value is never present in any exported type, log output, error message, or `HealthStatus`

2. **Given** `api_key_env: OPENAI_API_KEY` and the env var is NOT set (empty or missing)  
   **When** `LoadProfiles` is called  
   **Then** it returns `(nil, error)` with a message that names the provider and the missing env var name  
   **And** the error message does NOT contain any credential value

3. **Given** `api_key_env: ""` (empty string — e.g. a local unauthenticated provider)  
   **When** `LoadProfiles` is called  
   **Then** no env var lookup is attempted and the provider loads successfully with an empty key

## Tasks / Subtasks

- [ ] Extend `LoadProfiles` to resolve API keys after structural validation (AC: 1–3)
  - [ ] For each provider: if `APIKeyEnv != ""`, call `os.Getenv(APIKeyEnv)` (AC: 1, 3)
  - [ ] If resolved value is empty and `APIKeyEnv != ""`, return error naming provider + env var name (AC: 2)
  - [ ] Store resolved key in unexported field `resolvedAPIKey string` on internal provider runtime struct (AC: 1)
- [ ] Audit all exported types and log calls to confirm resolved key never escapes (AC: 1)
  - [ ] `HealthStatus` and `ProviderStatus` must not include any key-related fields
  - [ ] `slog.Info` startup line must not include key values
  - [ ] Error messages reference env var NAME only — never the resolved value (AC: 2)
- [ ] Add empty-`api_key_env` path: provider loads successfully with empty resolved key (AC: 3)

## Dev Notes

- API key resolution occurs in `LoadProfiles` immediately after structural validation passes (Story 1.2)
- Internal runtime struct (unexported `providerRuntime` or similar) holds resolved keys; this struct never appears in exported API surface
- `os.Getenv` is the ONLY place API keys are read — never at runtime in `Resolve` or `RecordUsage` (NFR-S3)
- Empty `api_key_env: ""` is valid — represents local providers (e.g. Ollama) that don't need a key
- Error format for missing env var: `"routing: config: provider %q api_key_env %q is not set"` — never include the resolved value
- The resolved key is used by gateway HTTP handlers (not the routing layer itself) — the `Router` interface does not expose it. The routing layer resolves it at startup and makes it available via the internal provider struct if gateway code needs to inject it into outbound requests. Exact interface for this is out of scope for this story; confirm with gateway architecture if needed.
- Security invariant: after this story, nothing must be able to extract key values from the `Router` interface or any type it returns

### Project Structure Notes

- File: `internal/routing/config.go` (extending `LoadProfiles` from Story 1.2)
- Internal runtime struct for providers: defined in `config.go` or a new unexported file — must NOT be in `types.go` (which is the exported types file)

### References

- [Source: docs/terminus/inference/provider-routing/prd.md#FR15, NFR-S3] — Env var key resolution; never read at runtime
- [Source: docs/terminus/inference/provider-routing/architecture.md#Additional Requirements] — "API key env-var names resolved by `LoadProfiles`; resolved keys never in any logged or exported type"
- [Source: docs/terminus/inference/provider-routing/epics.md#Story 1.3] — All AC scenarios

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
