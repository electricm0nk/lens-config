# Story 3.4: Add Smoke-Test Guidance and Failure Triage Notes in Runbook

Status: ready-for-dev

**Epic:** 3 — Verification and Release Integration
**Initiative:** terminus-infra-influxdb-monitoring
**Date:** 2026-04-12
**Depends on:** Stories 2.3, 3.1, 3.2

---

## Story

As an infrastructure operator,
I want smoke-test steps and failure triage notes added to the bootstrap runbook,
so that post-deployment validation and incident triage are covered in a single reference document.

---

## Context

**Repo:** `bmad.lens.projects`
**Docs location:** `docs/terminus/infra/influxdb-monitoring/runbook-bootstrap.md`
**Extension of:** Story 2.3 (which creates the runbook structure)

This story extends the bootstrap runbook with operational content for the verification phase: smoke tests to run after first deploy, and triage steps for the failure modes most likely to occur with InfluxDB on k3s.

---

## Technical Notes

**Smoke-test section additions:**

_After successful bootstrap (credentials in Vault, ESO synced, pod running):_

1. **UI access:** Port-forward or ingress to `http://influxdb.monitoring.svc.cluster.local:8086`
2. **Write test data point:**
   ```bash
   influx write \
     --bucket monitoring \
     --precision s \
     "smoke_test,source=operator value=1 $(date +%s)"
   ```
3. **Query test data back:**
   ```bash
   influx query 'from(bucket:"monitoring") |> range(start: -5m) |> filter(fn: (r) => r["_measurement"] == "smoke_test")'
   ```
4. **Expected:** Query returns at least one record with `value=1`

**Dev smoke test:**
Same commands, substitute `--bucket monitoring-dev` and dev service URL.

**Failure triage additions:**

| Symptom | Likely Cause | Resolution |
|---------|-------------|------------|
| Pod `CrashLoopBackOff` at startup | Missing/mis-keyed admin secret | Check `kubectl describe pod`, verify Secret key names match chart schema |
| ESO `SecretSyncError` | Vault path wrong or token expired | `kubectl describe externalsecret influxdb-admin -n monitoring` |
| PVC stuck `Pending` | Storage class unavailable | `kubectl describe pvc -n monitoring`, check storage class |
| Smoke test write fails | Token invalid or bucket not initialized | Confirm first-login was completed; re-run bootstrap |
| `influx` CLI reports wrong URL | Service port wrong or ingress not set | Check service port is 8086, confirm ingress config |

---

## Acceptance Criteria

1. Bootstrap runbook (`runbook-bootstrap.md`) includes a "Smoke Test" section with write + query steps
2. Smoke test section covers both prod (`monitoring`) and dev (`monitoring-dev`)
3. Failure triage table covers at minimum 5 failure modes (pod crash, ESO sync error, PVC pending, write fail, URL error)
4. All commands in runbook are tested/validated against the live cluster
5. Runbook update committed alongside Story 3.2 verify playbook

## Tasks / Subtasks

- [ ] Task 1: Read existing `runbook-bootstrap.md` from Story 2.3
  - [ ] Understand current structure
  - [ ] Identify natural insertion point for smoke test and triage sections

- [ ] Task 2: Write smoke test section
  - [ ] Port-forward or ingress access instructions
  - [ ] `influx write` test command for prod and dev
  - [ ] `influx query` verification command
  - [ ] Expected output description

- [ ] Task 3: Write failure triage section
  - [ ] Pod CrashLoopBackOff — secret key mismatch
  - [ ] ESO SecretSyncError — Vault path/token
  - [ ] PVC Pending — storage class
  - [ ] Write/query failure — token or bucket initialization
  - [ ] Any additional failures found during validation

- [ ] Task 4: Validate all commands against live cluster
  - [ ] Run smoke test on dev deployment
  - [ ] Confirm triage steps are accurate

- [ ] Task 5: Commit
  - [ ] `git commit -m "docs(monitoring): add smoke test and failure triage to bootstrap runbook"`
