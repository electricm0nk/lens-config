# Story 3.2: Add `verify-influxdb-deploy.yml` Ansible Playbook (Prod)

Status: done

**Epic:** 3 — Verification and Release Integration
**Initiative:** terminus-infra-influxdb-monitoring
**Date:** 2026-04-12
**Depends on:** Story 3.1
**Implemented:** this session

---

## Story

As an infrastructure operator,
I want a repeatable Ansible verify playbook for the InfluxDB prod deployment,
so that prod health can be validated after upgrades or changes with a consistent, automated check.

---

## Context

**Repo:** `terminus.infra`
**File:** `platforms/k3s/ansible/playbooks/verify-influxdb-deploy.yml`
**Pattern:** Mirrors the dev playbook from Story 3.1 — same checks, prod URL
**Runner:** Semaphore project 2 (`terminus-infra-ops`) — infra ops, not Temporal-triggered

**cicd.md constraints (non-negotiable):**
- Semaphore runners do NOT have `kubectl` available — HTTP/API checks only
- Use external DNS `http://influxdb.trantor.internal` NOT `svc.cluster.local`
- `validate_certs: false` for `.trantor.internal`

---

## Technical Notes

**Verify checks (prod):**
1. InfluxDB `/health` endpoint returns `{"status":"pass"}` at `http://influxdb.trantor.internal/health`
2. HTTP response code is 200
3. Response body contains `status: pass`

**Differences from dev playbook:**
- `health_url: "http://influxdb.trantor.internal/health"` (not `-dev`)
- Prod is expected more stable; fewer retries acceptable but keep 5/15s for safety

**Prod-specific considerations:**
- Strictly read-only: no writes to InfluxDB
- No kubectl, no svc.cluster.local

---

## Acceptance Criteria

1. `playbooks/verify-influxdb-deploy.yml` exists in terminus.infra
2. Playbook uses `ansible.builtin.uri` HTTP check against `http://influxdb.trantor.internal/health`
3. No `kubectl` or cluster-internal DNS in the playbook
4. `validate_certs: false` is set
5. Playbook is strictly read-only
6. Playbook exits non-zero if health check fails
7. Pattern is parallel to dev playbook (same structure, different URL)

## Tasks / Subtasks

- [ ] Task 1: Copy and adapt dev playbook for prod
  - [ ] Copy `verify-influxdb-dev-deploy.yml` as `verify-influxdb-deploy.yml`
  - [ ] Update `health_url: "http://influxdb.trantor.internal/health"`
  - [ ] Confirm no kubectl or svc.cluster.local references remain

- [ ] Task 2: Test playbook
  - [ ] Run against prod target
  - [ ] Confirm health check passes and no destructive operations

- [ ] Task 3: Commit
  - [ ] `git commit -m "feat(monitoring): add verify-influxdb-deploy prod playbook"`
