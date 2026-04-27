# Story: sales-2-01

## Context

**Feature:** fourdogs-etailpet-api-sales
**Sprint:** 2 — Deploy and Validate
**Priority:** high
**Estimate:** M

## User Story

As an operator, I want a Helm chart and ArgoCD Application for `fourdogs-etailpet-sales-trigger`,
so that the sales worker can be deployed to k3s via GitOps using the same conventions as the
transactions worker.

## Acceptance Criteria

- [ ] Helm chart at `deploy/helm/fourdogs-etailpet-sales-trigger/` in `fourdogs-central` repo
- [ ] Chart includes: `Deployment` (replicas: 1, no HPA), `ServiceAccount`, env var wiring from `Secret`
- [ ] `TRIGGER_ENABLED` defaults to `false` in `values.yaml`
- [ ] `TRIGGER_INTERVAL_HOURS` defaults to `24` in `values.yaml`
- [ ] Docker image built from `cmd/etailpet-sales-trigger/`; pushed to GHCR as `ghcr.io/fourdogs/etailpet-sales-trigger:{tag}`; CI build step added
- [ ] ArgoCD Application manifest references `fourdogs` project and `fourdogs-etailpet-sales-trigger` namespace
- [ ] Deployment has a liveness probe (match emailfetcher / transactions worker pattern)
- [ ] `helm lint deploy/helm/fourdogs-etailpet-sales-trigger/` passes with no warnings

## Technical Notes

Use `deploy/helm/fourdogs-etailpet-trigger/` as the direct template. The chart structures are
identical; only the following differ:

| Field | transactions | sales |
|---|---|---|
| Chart name | `fourdogs-etailpet-trigger` | `fourdogs-etailpet-sales-trigger` |
| Image | `etailpet-trigger` | `etailpet-sales-trigger` |
| Namespace | `fourdogs-etailpet-trigger` | `fourdogs-etailpet-sales-trigger` |
| Secret name | `fourdogs-etailpet-api-secrets` | `fourdogs-etailpet-api-sales-secrets` |
| Additional env vars | `ETAILPET_BASE_URL` | `ETAILPET_LEGACY_BASE_URL`, `TRIGGER_LOOKBACK_DAYS` |

**Additional values.yaml entries for sales chart:**
```yaml
etailpetLegacyBaseUrl: "https://www.etailpet.com"
etailpetSchemaName: "fourdogspetsupplies"
etailpetTimezone: "America/Los_Angeles"
triggerReportType: "sales-line"
triggerFileFormat: "xlsx"
triggerLookbackDays: 7
```

**Namespace:** Create a new namespace `fourdogs-etailpet-sales-trigger` in the k3s manifest or
via Helm chart namespace creation (match the transactions worker convention).

## Dependencies

**Blocked by:** sales-1-02 (binary must exist to build image)
**Blocks:** sales-2-03

## Definition of Done

- [ ] `helm lint` passes
- [ ] `helm template` output reviewed and correct
- [ ] ArgoCD Application manifest committed to GitOps repo
- [ ] CI image build step added and confirmed working
- [ ] Story acceptance criteria verified by author
- [ ] PR reviewed and merged to `fourdogs-etailpet-api-sales` branch
