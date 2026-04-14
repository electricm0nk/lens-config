# Story 4.1: Resolve emits structured log per request

Status: ready-for-dev

## Story

As an **operator**,
I want every `Resolve` call to emit a structured `slog.Debug` log line with provider, workload class, and degradation status,
So that I can observe routing decisions in production logs without any PII leakage.

## Acceptance Criteria

1. **Given** `Resolve` successfully returns a provider  
   **When** the request is processed  
   **Then** an `slog.DebugContext` line is emitted containing: selected provider name, requested profile name, `degraded: false`  
   **And** the log line does NOT contain prompt text, response text, API key values, or caller identity tokens

2. **Given** `Resolve` selects a fallback provider due to exhaustion  
   **When** the request is processed  
   **Then** an `slog.DebugContext` line is emitted containing: fallback provider name, profile name, `degraded: true`, `degradation_reason`  
   **And** the log line contains no PII or credential values

3. **Given** `Resolve` returns `ErrNoEligibleProvider`  
   **When** the request is processed  
   **Then** an `slog.WarnContext` line is emitted naming the profile and that no provider was available  
   **And** no sensitive values appear in the log

## Tasks / Subtasks

- [ ] Add `slog.DebugContext` call after successful provider selection in `Resolve` (AC: 1, 2)
  - [ ] Success: `slog.DebugContext(ctx, "routing: resolved", "provider", resp.Provider, "profile", req.Profile, "degraded", resp.Degraded)`
  - [ ] Degraded: include `"degradation_reason", resp.DegradationReason` when `resp.Degraded == true` (AC: 2)
- [ ] Add `slog.WarnContext` call when returning `ErrNoEligibleProvider` (AC: 3)
  - [ ] `slog.WarnContext(ctx, "routing: no eligible provider", "profile", req.Profile)`
- [ ] Audit log fields to confirm allow-list compliance (AC: 1, 2, 3):
  - [ ] Allowed: `provider` name, `profile` name, `degraded` bool, `degradation_reason` string, token counts
  - [ ] NEVER allowed: prompt content, response content, API key values, caller identity
- [ ] Write log PII-safety test (preview for Story 4.3 — can be deferred to that story)

## Dev Notes

- Logging allow-list (from architecture): "provider name, workload class, token counts, degradation, budget transitions, config path" — everything else is forbidden
- `slog.DebugContext` uses the context to propagate trace IDs if the gateway sets them — always use `DebugContext`/`WarnContext` (with ctx), never `slog.Debug` (without ctx)
- No logger field on the `router` struct — use `slog.Default()` context-aware calls. If the gateway injects a custom logger, the architecture must be extended; for MVP use `slog` package-level functions with `ctx`
- Error path log added here for the "no eligible provider" case — the panic recovery log was added in Story 2.3 and stays there
- Do NOT log the `requestBody` — not even its length in the Debug line (length is only logged in the `slog.Warn` for nil-usage estimate in RecordUsage)

### Project Structure Notes

- File: `internal/routing/router.go` (extending `Resolve` exits)
- Three logging sites in `Resolve`: success debug, degraded debug, no-provider warn

### References

- [Source: docs/terminus/inference/provider-routing/prd.md#FR18, FR19, NFR-S1, NFR-S2] — Log per request; never log PII
- [Source: docs/terminus/inference/provider-routing/architecture.md#Additional Requirements] — Logging allow-list
- [Source: docs/terminus/inference/provider-routing/epics.md#Story 4.1] — All AC scenarios

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
