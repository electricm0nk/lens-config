# Story: transactions-2-03

## Context

**Feature:** transactions
**Sprint:** 2 — Deploy & Validate
**Priority:** high
**Estimate:** S

## User Story

As an operator, I want to validate that the deployed `fourdogs-etailpet-trigger` worker fires a
live trigger to the eTailPet API in staging and that the resulting email is delivered to the
emailfetcher inbox, so that the end-to-end path is confirmed before production enable.

## Acceptance Criteria

- [ ] Worker deployed to staging k3s cluster in namespace `fourdogs-central`; ArgoCD reports `Synced / Healthy`
- [ ] `TRIGGER_ENABLED=false` initially; validate worker starts and logs `trigger_disabled` each tick
- [ ] `TRIGGER_ENABLED=true` set in Helm values and applied via ArgoCD sync; next tick fires live trigger
- [ ] Worker logs show: `trigger_attempt` → `trigger_success` (status code 200 or 2xx) within expected latency
- [ ] Email from eTailPet received in emailfetcher inbox within expected delivery window (to be confirmed during spike)
- [ ] Worker logs contain no credential values
- [ ] Operator runbook written at `docs/fourdogs/etailpet-api/transactions/runbook.md`:
  - How to enable / disable trigger (`TRIGGER_ENABLED` Helm value)
  - How to check trigger worker health (`kubectl logs`, `kubectl get pod`)
  - How to force an immediate trigger without waiting for the tick (restart pod + `--dry-run=false`)
  - How to confirm delivery: check emailfetcher inbox logs for expected email metadata
  - How to roll back: set `TRIGGER_ENABLED=false` + ArgoCD sync
  - Alert: what to watch for in `trigger_exhausted` log events

## Technical Notes

**Forcing an immediate trigger for validation:**
Option A: Restart the pod and wait one full tick interval; confirm `trigger_scheduled` log appears.
Option B: `kubectl rollout restart deploy/fourdogs-etailpet-trigger -n fourdogs-central` — clean
  restart, ticker fires after one configured interval.
Option C: Temporarily set `TRIGGER_INTERVAL_HOURS=1` (1 hour) in values.yaml for staging to reduce
  wait time without risking rate limits.

**Note:** Do NOT use `TRIGGER_INTERVAL_HOURS` values smaller than what Sprint 0 confirms is safe.
A very short interval risks hitting the eTailPet rate limit or triggering an IP block before the
rate limit is documented. Use Option C (1-hour interval) as the minimum for staging validation.

**Pod health check commands:**
```bash
kubectl get pod -n fourdogs-central -l app=fourdogs-etailpet-trigger
kubectl logs -n fourdogs-central deploy/fourdogs-etailpet-trigger --follow
kubectl describe pod -n fourdogs-central -l app=fourdogs-etailpet-trigger
```

**Rollback:**
```bash
# Disable trigger immediately
helm upgrade fourdogs-etailpet-trigger deploy/helm/fourdogs-etailpet-trigger \
  --set triggerEnabled=false -n fourdogs-central
# Or via ArgoCD: update values.yaml and sync
```

**Runbook file path:** `docs/fourdogs/etailpet-api/transactions/runbook.md`

## Dependencies

**Blocked by:** transactions-2-02 (credentials must be injected before live trigger)
**Blocks:** feature close (this is the final acceptance story)

## Definition of Done

- [ ] Code compiles with no errors or warnings (`go build ./...`)
- [ ] `go test ./...` passes
- [ ] `go vet ./...` passes
- [ ] No credentials, tokens, or secrets in code or committed files
- [ ] Structured log events fire correctly for the story's scope
- [ ] Story acceptance criteria verified by author
- [ ] PR reviewed (or self-reviewed for solo work) and merged to `transactions` branch

## Open Questions

- Expected email delivery window from eTailPet after trigger: unknown until spike.
  Document confirmed window in runbook.
