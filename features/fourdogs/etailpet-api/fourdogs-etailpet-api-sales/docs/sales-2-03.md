# Story: sales-2-03

## Context

**Feature:** fourdogs-etailpet-api-sales
**Sprint:** 2 — Deploy and Validate
**Priority:** high
**Estimate:** S

## User Story

As an operator, I want the `fourdogs-etailpet-sales-trigger` worker validated in staging with a
live trigger and a written runbook, so that I can operate the service with confidence in production.

## Acceptance Criteria

- [ ] Worker deployed to staging with `TRIGGER_ENABLED=false` initially; pod running and healthy
- [ ] `TRIGGER_ENABLED` set to `true` and `TRIGGER_INTERVAL_HOURS` set to `1` for staging validation
- [ ] Live trigger fires; `trigger_success` logged with `status_code: 200`
- [ ] Sales-line xlsx email received at Four Dogs retailer inbox (confirmed by operator)
- [ ] emailfetcher confirms receipt of the attachment (existing routing rule works OR new routing rule documented and applied)
  - **Cross-service dependency:** If a new emailfetcher routing rule is needed, open a separate [emailfetcher] routing rule ticket. Adding the rule is NOT in scope for this feature — it requires a separate story in the emailfetcher feature. Do not block this story on routing rule review; document the gap and escalate.
- [ ] Worker reverted to `TRIGGER_INTERVAL_HOURS: 24` and production config before merge
- [ ] Operator runbook written at `docs/fourdogs/etailpet-api/fourdogs-etailpet-api-sales/runbook.md`
- [ ] Runbook covers: pod health commands, log event reference, enable/disable procedure, force-trigger procedure

## Technical Notes

**Staging validation sequence:**

1. Deploy with `TRIGGER_ENABLED=false`; confirm pod is Running
2. Confirm ESO secret synced: `kubectl get secret fourdogs-etailpet-api-sales-secrets -n fourdogs-etailpet-sales-trigger`
3. Set `TRIGGER_INTERVAL_HOURS=1` and `TRIGGER_ENABLED=true` via values override; sync ArgoCD
4. Wait for first tick (fires at pod startup); check logs: `kubectl logs -n fourdogs-etailpet-sales-trigger ...`
5. Confirm `trigger_success status_code=200` in logs
6. Check retailer inbox for eTailPet sales-line report email
7. Check emailfetcher logs for receipt of the attachment
8. Revert to production config before merging

**Do NOT use `TRIGGER_INTERVAL_HOURS=0.01` or sub-1h intervals** — risk of rate limit exhaustion
(rate limit cap on `www.etailpet.com` is undocumented; stay conservative).

**emailfetcher routing:** If the emailfetcher uses subject-line filtering and the sales-line email
subject differs from transaction-report subjects (compare to subjects documented in sales-0-01
spike), add a routing rule to the emailfetcher config in this story.

**Runbook template** (adapt from transactions runbook at `docs/fourdogs/etailpet-api/transactions/runbook.md`):
- Pod health commands section
- Log event reference table (all `trigger_*` events)
- Enable/disable procedure
- Force-trigger procedure (restart deployment)
- Incident response: what to do if `trigger_exhausted` fires

## Dependencies

**Blocked by:** sales-2-01 (Helm chart + ArgoCD must be deployed), sales-2-02 (ESO secret must exist)
**Blocks:** none

## Definition of Done

- [ ] Live trigger confirmed in staging (`trigger_success` logged)
- [ ] Sales-line email confirmed in retailer inbox
- [ ] emailfetcher confirms attachment receipt (or routing rule added and tested)
- [ ] Worker reverted to production config (`TRIGGER_INTERVAL_HOURS: 24`, `TRIGGER_ENABLED: true`)
- [ ] Runbook written and committed
- [ ] Story acceptance criteria verified by author
- [ ] PR reviewed and merged to `fourdogs-etailpet-api-sales` branch
