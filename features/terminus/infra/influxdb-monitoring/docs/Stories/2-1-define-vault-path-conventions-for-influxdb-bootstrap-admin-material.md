# Story 2.1: Define Vault Path Conventions for InfluxDB Bootstrap/Admin Material

Status: done

**Epic:** 2 — Secrets and Bootstrap Hardening
**Initiative:** terminus-infra-influxdb-monitoring
**Date:** 2026-04-12
**Depends on:** Story 1.4
**Implemented:** this session

---

## Story

As an infrastructure operator,
I want clearly defined Vault KV v2 path conventions for InfluxDB admin credentials,
so that bootstrap secrets are stored consistently and can be reliably wired by ESO.

---

## Context

**Repo:** `terminus.infra` (docs/runbook) and Vault itself (out-of-band write)
**Vault engine:** KV v2
**Pattern:** Follows existing Vault path conventions used by other terminus services

InfluxDB requires an initial admin token and/or username/password for first-time setup. These cannot be stored in git — they are injected via Vault + ESO during deployment. This story defines the path schema and automation contract consumed by `seed-influxdb-secrets.yml`.

---

## Technical Notes

**Proposed Vault path schema:**

```
secret/terminus/monitoring/influxdb/admin
  token:     <admin token>
  username:  <admin username>
  password:  <admin password>

secret/terminus/monitoring/influxdb-dev/admin
  token:     <admin token>
  username:  <admin username>
  password:  <admin password>
```

**Key decisions:**
- Separate paths for prod (`monitoring`) and dev (`monitoring-dev`) admin material
- No cross-environment secret sharing
- Path prefix `terminus/monitoring/` aligns with namespace naming
- `token` field is the preferred InfluxDB v2 auth token (used in values/env injection)

**Helm chart auth injection approach:**
- InfluxDB Helm chart (`influxdb2`) supports `adminUser.password` via values or via existing secret
- Use `adminUser.existingSecret` pattern to reference a k8s Secret created by ESO
- Secret keys must match chart expectations: check `influxdb2` chart schema for exact key names

---

## Acceptance Criteria

1. Vault path schema for prod and dev admin material is documented
2. Path naming conventions are consistent with existing terminus Vault conventions
3. Path schema is validated against the `influxdb2` Helm chart `adminUser.existingSecret` requirements
4. Vault paths are written to the operator bootstrap runbook (preview for Story 2.3)
5. No credentials committed to git

## Tasks / Subtasks

- [ ] Task 1: Research `influxdb2` chart secret schema
  - [ ] Read `influxdb2` chart `values.yaml` — identify `adminUser.existingSecret` key requirements
  - [ ] Document expected k8s Secret key names (e.g., `admin-password`, `admin-token`)

- [ ] Task 2: Define Vault path schema
  - [ ] Confirm path prefix aligns with existing terminus Vault structure
  - [ ] Define prod path: `secret/terminus/monitoring/influxdb/admin`
  - [ ] Define dev path: `secret/terminus/monitoring/influxdb-dev/admin`
  - [ ] Document field names mapping: Vault field → ESO target key → k8s Secret key

- [x] Task 3: Implement pipeline seeding (no manual Vault writes)
  - [x] Added `platforms/k3s/ansible/playbooks/seed-influxdb-secrets.yml`
  - [x] Playbook preserves existing values and generates missing values idempotently
  - [x] Writes prod and dev paths via Vault API

- [x] Task 4: Wire into pipeline task runner
  - [x] Added `seed-influxdb-secrets` Semaphore template in `tofu/environments/semaphore/main.tf`
  - [x] Uses survey var `vault_token` to satisfy release/seed contract
