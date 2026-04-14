---
verdict: PASS_WITH_NOTES
initiative: terminus-inference-provider-routing
audience: medium
reviewedAt: '2026-04-06'
reviewedBy: [analyst, architect, pm, qa]
artifacts:
  - docs/terminus/inference/provider-routing/prd.md
  - docs/terminus/inference/provider-routing/architecture.md
---

# Adversarial Review — terminus-inference-provider-routing

**Entry gate for:** small → medium promotion
**Mode:** Party (Inquisitor Greyfax · Perturabo · Lord Commander Creed · Watch-Captain Artemis)
**Verdict: PASS_WITH_NOTES**

All noted gaps are resolvable in devproposal stories. No findings require rework of the PRD or architecture before advancing.

---

## Findings

### Important — Resolve in DevProposal Stories

**F1 — Caller workload class conveyance undefined**
*Greyfax:* FR1 states the gateway resolves "by passing workload class and caller identity to the router." `RoutingRequest` is defined but neither document states *how* a caller specifies which workload class they want. HTTP header? Request body field? Hardcoded per-endpoint? The implementing agent will make this up without an explicit decision.
→ **Resolution path:** Add a story to devproposal that specifies the HTTP API contract for workload class selection.

**F2 — Caller identity absent from RoutingRequest**
*Greyfax:* FR1 includes "caller identity" in the resolution contract. NFR-S4 states it comes from the gateway auth middleware context. `RoutingRequest` has no `CallerID` field and there is no explicit statement that caller identity is intentionally omitted from MVP routing decisions. This is either a gap or an unstated decision.
→ **Resolution path:** Architecture should be amended with a one-line clarification: caller identity available in `context.Context` but NOT used for provider selection in MVP (deferred to rate-guardrail phase). OR add `CallerID string` to `RoutingRequest`.

**F3 — Budget reset period anchor undefined**
*Greyfax:* FR8 says "daily" budget. FR13 says state resets on process restart (no persistence). There is no definition of when the daily period resets independently of process restarts — UTC midnight, process-start-relative, or configurable. Two deployments with different uptime patterns will behave inconsistently.
→ **Resolution path:** Add a story decision: for MVP, "daily" is process-start-relative (budget resets on restart). Hard-coded; no clock-based reset. Document this explicitly in architecture.

**F4 — Token fallback estimate formula absent**
*Greyfax:* Domain requirements state: "fall back to estimated count from request body character length; log warning." Neither document defines the estimation formula. The implementing agent will invent it.
→ **Resolution path:** Add a story specifying: `estimated_tokens = len(request_body_json) / 4` (industry heuristic). Surface as `slog.Warn` on the `RecordUsage` call site.

**F5 — ROUTING_CONFIG_PATH empty-string behavior undefined**
*Perturabo:* `os.Getenv("ROUTING_CONFIG_PATH")` with no specified fallback. A misconfigured deployment must fail loudly, not quietly.
→ **Resolution path:** Add to architecture: if `ROUTING_CONFIG_PATH` is empty, `LoadProfiles` returns `fmt.Errorf("routing: ROUTING_CONFIG_PATH environment variable is not set")`. No default path.

**F6 — Schema version value never declared**
*Perturabo:* Architecture mandates "unsupported version → hard error" but never states what the *current supported version* is. Agents will choose different values; config fixture YAML will embed different values.
→ **Resolution path:** Add to architecture: `schema_version: 1` is the only supported value at MVP. Constant: `SupportedSchemaVersion = 1`.

**F7 — Config YAML structure never fully specified**
*Perturabo:* Field names appear across the architecture but no canonical YAML schema is shown. The PRD has a partial schema in the API surface section. The architecture should resolve this with a single authoritative YAML example.
→ **Resolution path:** Add a YAML example block to the architecture's config section showing the complete schema.

**F9 — HealthStatus and ProviderStatus shapes undefined**
*Lord Commander Creed:* `HealthCheck` returns `HealthStatus`. FR20 specifies required fields (`budget_used_usd`, `budget_limit_usd`, `budget_exhausted`) but neither document shows the struct definition. Implementing agent will invent field names; gateway handler will have to match.
→ **Resolution path:** Add struct definitions to architecture or `types.go` spec section.

---

### Notes — Address at Implementation Discretion

**F8 — ErrInvalidWorkloadClass missing from integration point code snippet**
*Perturabo:* The `internal/errors` code example in integration points only shows `ErrNoEligibleProvider → 503`. The `ErrInvalidWorkloadClass → 400` mapping is in prose only. Add it to the code block.

**F10 — Degradation reason values should be constants, not raw strings**
*Lord Commander Creed:* `DegradationReason` string values ("budget_exhausted", "provider_unhealthy") should be declared as typed constants in `errors.go` or `types.go` to prevent string drift.

**F11 — testdata directory policy should be explicitly resolved**
*Artemis:* The patterns section says "no testdata/ unless fixtures require files." Config tests necessarily require YAML fixtures. Explicitly state that `internal/routing/testdata/` holds YAML fixtures for `config_test.go`.

**F12 — Race tests must be run with -race**
*Artemis:* Budget concurrency tests should explicitly state `-race` is required in CI. Without it, data races pass logically but fail in production under concurrent load.

---

## Disposition

| Finding | Priority | Disposition |
|---|---|---|
| F1 Workload class conveyance | Important | Devproposal story |
| F2 Caller identity in RoutingRequest | Important | Architecture amendment (one-liner) |
| F3 Budget reset anchor | Important | Devproposal story decision |
| F4 Token fallback formula | Important | Devproposal story |
| F5 Empty ROUTING_CONFIG_PATH | Important | Architecture amendment |
| F6 Schema version value | Important | Architecture amendment |
| F7 Config YAML schema | Important | Architecture amendment |
| F9 HealthStatus shape | Important | Architecture amendment |
| F8 ErrInvalidWorkloadClass in snippet | Minor | Implementation note |
| F10 Degradation reason constants | Minor | Implementation note |
| F11 testdata directory | Minor | Implementation note |
| F12 Race test flag | Minor | Implementation note |
