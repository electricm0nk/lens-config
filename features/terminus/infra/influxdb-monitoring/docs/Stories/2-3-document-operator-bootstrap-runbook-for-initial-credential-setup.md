# Story 2.3: Document Operator Bootstrap Runbook for Initial Credential Setup

Status: ready-for-dev

**Epic:** 2 — Secrets and Bootstrap Hardening
**Initiative:** terminus-infra-influxdb-monitoring
**Date:** 2026-04-12
**Depends on:** Stories 2.1, 2.2

---

## Story

As an infrastructure operator,
I want a concise bootstrap runbook for InfluxDB credential setup,
so that any operator can bring up a fresh InfluxDB instance from scratch without tribal knowledge.

---

## Context

**Repo:** `bmad.lens.projects`
**Docs location:** `docs/terminus/infra/influxdb-monitoring/runbook-bootstrap.md`

The runbook covers the one-time setup steps that cannot be automated: writing Vault credentials, validating ESO delivery, and verifying InfluxDB first login. It is the operational companion to the automated GitOps + ESO wiring in Stories 2.1 and 2.2.

---

## Technical Notes

**Runbook sections:**

1. **Prerequisites** — cluster access, Vault access, `kubectl`, `argocd` CLI, `vault` CLI
2. **Vault credential write** — exact `vault kv put` commands for prod and dev paths
3. **ESO sync check** — `kubectl get externalsecret -n monitoring`, verify `Ready` status
4. **k8s Secret verification** — `kubectl get secret influxdb-admin -n monitoring -o jsonpath=...`
5. **InfluxDB first login** — port-forward or ingress URL, verify login with bootstrapped credentials
6. **Smoke test** — write one data point, read it back via `influx` CLI or UI
7. **Failure triage** — common failure modes:
   - ESO `SecretSyncError`: check Vault path, Vault token permissions
   - Pod `CrashLoopBackOff`: check secret key name mismatch against chart schema
   - PVC `Pending`: check storage class availability

---

## Acceptance Criteria

1. `docs/terminus/infra/influxdb-monitoring/runbook-bootstrap.md` exists
2. Runbook covers prod and dev separately where paths/commands differ
3. All `vault kv put` commands use correct paths from Story 2.1
4. ESO sync verification steps are included
5. Failure triage section covers at minimum: ESO sync error, pod crash on startup, PVC pending
6. Runbook is self-contained (no undocumented dependencies on tribal knowledge)

## Tasks / Subtasks

- [ ] Task 1: Draft runbook outline
  - [ ] Verify Vault path names from Story 2.1 doc
  - [ ] Verify ESO ExternalSecret names from Story 2.2

- [ ] Task 2: Write prerequisites section
  - [ ] List required tools and access
  - [ ] Add Vault auth method note (approle or token — match cluster config)

- [ ] Task 3: Write Vault credential write section
  - [ ] Add verbatim `vault kv put` commands for prod and dev
  - [ ] Add note: credentials are operator-chosen on first bootstrap; store in 1Password or equivalent

- [ ] Task 4: Write ESO + k8s Secret verification section
  - [ ] Add `kubectl get externalsecret` command
  - [ ] Add `kubectl describe externalsecret` for troubleshooting
  - [ ] Add `kubectl get secret influxdb-admin -n monitoring` verification step

- [ ] Task 5: Write InfluxDB first-login and smoke test section
  - [ ] Add port-forward command for headless access
  - [ ] Describe UI first-login flow (or `influx` CLI setup command)
  - [ ] Add simple data write + query smoke test

- [ ] Task 6: Write failure triage section
  - [ ] Cover ESO `SecretSyncError`
  - [ ] Cover pod `CrashLoopBackOff` on startup
  - [ ] Cover PVC `Pending`

- [ ] Task 7: Commit
  - [ ] `git commit -m "docs(monitoring): add influxdb operator bootstrap runbook"`
