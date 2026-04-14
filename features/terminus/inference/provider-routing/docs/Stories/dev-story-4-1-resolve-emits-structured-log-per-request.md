# Story 4.1: Resolve emits structured log per request

**Initiative:** terminus-inference-provider-routing
**Epic:** 4 — Operator can observe routing health
**Status:** review

## Story

As an **operator**,
I want every `Resolve` call to emit a structured `slog.Debug` log line with provider, workload class, and degradation status,
So that I can observe routing decisions in production logs without any PII leakage.

## Acceptance Criteria

**Given** `Resolve` successfully returns a provider
**When** the request is processed
**Then** an `slog.DebugContext` line is emitted containing: selected provider name, requested profile name, `degraded: false`
**And** the log line does NOT contain prompt text, response text, API key values, or caller identity tokens

**Given** `Resolve` selects a fallback provider due to exhaustion
**When** the request is processed
**Then** an `slog.DebugContext` line is emitted containing: fallback provider name, profile name, `degraded: true`, `degradation_reason`
**And** the log line contains no PII or credential values

**Given** `Resolve` returns `ErrNoEligibleProvider`
**When** the request is processed
**Then** an `slog.WarnContext` line is emitted naming the profile and that no provider was available
**And** no sensitive values appear in the log

## Technical Notes

- Target repo: `terminus-inference-gateway`
- Add `slog.DebugContext` calls inside `Resolve` in `router.go`
- Slog allow-list — ONLY these keys may appear in routing log lines:
  - `provider` (selected provider name)
  - `profile` (workload class)
  - `degraded` (bool)
  - `fallback_provider` (string, only when degraded)
  - `degradation_reason` (string, only when degraded)
  - `estimated_tokens` (int, only in RecordUsage nil-usage path)
  - Budget transition values (`budget_used`, `budget_limit` — numbers only)
- NEVER log: prompt text, response text, API key values, full request body, caller identity

## Dependencies

- Story 2.3 (Resolve error paths)
- Story 3.3 (degradation/fallback paths)
