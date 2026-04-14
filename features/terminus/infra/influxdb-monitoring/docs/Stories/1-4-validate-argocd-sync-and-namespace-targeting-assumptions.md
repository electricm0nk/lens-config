# Story 1.4: Validate ArgoCD Sync Behavior and Namespace Targeting Assumptions

Status: ready-for-dev

**Epic:** 1 — GitOps InfluxDB Deployment Foundation
**Initiative:** terminus-infra-influxdb-monitoring
**Date:** 2026-04-12
**Depends on:** Stories 1.1, 1.2, 1.3

---

## Story

As an infrastructure operator,
I want to confirm that the InfluxDB ArgoCD applications sync correctly and target the right namespaces,
so that I can verify the GitOps foundation is working before proceeding with secrets hardening.

---

## Context

**Repo:** `terminus.infra` (ArgoCD apps already committed)
**Check target:** Live k3s cluster — ArgoCD UI or `argocd app` CLI
**Namespaces to verify:** `monitoring` (prod), `monitoring-dev` (dev)

This is a validation story — no new code is written. The goal is to confirm that the multi-source ArgoCD apps deployed in 1.1–1.3 are syncing as expected on the cluster.

> **Note:** InfluxDB may not fully initialize until Epic 2 (Vault/ESO credentials) is complete. A `Degraded` health status due to missing auth secret is acceptable at this stage; the key check is `Synced` status and namespace existence.

---

## Technical Notes

**Check 1 — ArgoCD app status:**
```bash
argocd app get influxdb
argocd app get influxdb-dev
```
Expected: `Sync Status: Synced`, `Health Status: Healthy` (or `Degraded` if secrets not bootstrapped)

**Check 2 — Namespace existence:**
```bash
kubectl get namespace monitoring
kubectl get namespace monitoring-dev
```
Expected: both namespaces present with `STATUS: Active`

**Check 3 — Pod state:**
```bash
kubectl get pods -n monitoring
kubectl get pods -n monitoring-dev
```
Expected: InfluxDB pods present (may be in `Pending` or `CrashLoopBackOff` if secrets not yet bootstrapped)

**Check 4 — PVC provisioning:**
```bash
kubectl get pvc -n monitoring
kubectl get pvc -n monitoring-dev
```
Expected: PVCs bound to storage class

**If pods crash on startup:** This is expected if Vault/ESO is not yet bootstrapped. Document any chart key issues found (e.g., unexpected config key names) — this feeds Epic 2 and the `.todo` tracking file.

---

## Acceptance Criteria

1. ArgoCD reports both `influxdb` and `influxdb-dev` apps as `Synced`
2. `monitoring` and `monitoring-dev` namespaces exist in the cluster
3. InfluxDB pods are scheduled (even if not yet `Running` due to missing secrets)
4. PVCs are bound with expected storage sizes (50Gi prod, 10Gi dev)
5. Any chart key mismatches or config drift issues are documented in `docs/terminus/.todo/`

## Tasks / Subtasks

- [ ] Task 1: Check ArgoCD sync status for both apps
  - [ ] Run `argocd app get influxdb` — document output
  - [ ] Run `argocd app get influxdb-dev` — document output
  - [ ] If `OutOfSync`: inspect diff and correct values file keys

- [ ] Task 2: Verify namespace creation
  - [ ] Confirm `monitoring` namespace active
  - [ ] Confirm `monitoring-dev` namespace active

- [ ] Task 3: Verify pod scheduling
  - [ ] Confirm pods scheduled in both namespaces
  - [ ] Note any `CrashLoopBackOff` root cause (expected if pre-secrets)

- [ ] Task 4: Verify PVC binding
  - [ ] Confirm `monitoring` PVC bound at 50Gi
  - [ ] Confirm `monitoring-dev` PVC bound at 10Gi

- [ ] Task 5: Document findings
  - [ ] If chart key drift found: add item to `docs/terminus/.todo/2026-04-12-influxdb-monitoring-bootstrap.md`
  - [ ] Update story status to `done`
