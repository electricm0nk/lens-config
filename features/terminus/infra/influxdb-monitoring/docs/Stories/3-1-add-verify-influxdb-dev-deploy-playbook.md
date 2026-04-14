# Story 3.1: Add `verify-influxdb-dev-deploy.yml` Ansible Playbook

Status: done

**Epic:** 3 — Verification and Release Integration
**Initiative:** terminus-infra-influxdb-monitoring
**Date:** 2026-04-12
**Depends on:** Stories 1.4, 2.2
**Implemented:** this session

---

## Story

As an infrastructure operator,
I want a repeatable Ansible verify playbook for the InfluxDB dev deployment,
so that I can validate the dev environment is healthy after any change without manual cluster probing.

---

## Context

**Repo:** `terminus.infra`
**File:** `platforms/k3s/ansible/playbooks/verify-influxdb-dev-deploy.yml`
**Pattern:** Follows existing verify playbooks (e.g., `verify-inference-gateway-dev-deploy.yml`)
**Runner:** Semaphore project 2 (`terminus-infra-ops`) — InfluxDB is infra ops, NOT a Temporal-triggered release service

**cicd.md constraints (non-negotiable):**
- Semaphore runners do NOT have `kubectl` or `argocd` CLI available — use HTTP/API checks only
- Use external DNS `http://influxdb-dev.trantor.internal` NOT cluster-internal `svc.cluster.local`
- Verify Story 1.5 (ingress + DNS) is complete before this story begins — without the Traefik ingress the HTTP check has no target
- Use `validate_certs: false` for `.trantor.internal` (self-signed cert)

---

## Technical Notes

**Verify checks (dev):**
1. InfluxDB `/health` endpoint returns `{"status":"pass"}` at `http://influxdb-dev.trantor.internal/health`
2. HTTP response code is 200
3. Response body contains `status: pass`

**Playbook structure:**
```yaml
- name: Verify InfluxDB Dev Deployment
  hosts: localhost
  gather_facts: false
  vars:
    health_url: "http://influxdb-dev.trantor.internal/health"
  tasks:
    - name: Check InfluxDB dev health endpoint
      ansible.builtin.uri:
        url: "{{ health_url }}"
        method: GET
        return_content: true
        validate_certs: false
        status_code: 200
      register: health_response
      retries: 5
      delay: 15
      until: health_response.status == 200

    - name: Assert health response is pass
      ansible.builtin.assert:
        that:
          - "health_response.json.status == 'pass'"
        fail_msg: "InfluxDB dev health check failed: {{ health_response.content }}"
```

**No kubectl. No svc.cluster.local.** Semaphore runners are not on cluster nodes.

---

## Acceptance Criteria

1. `playbooks/verify-influxdb-dev-deploy.yml` exists in terminus.infra
2. Playbook uses `ansible.builtin.uri` HTTP check against `http://influxdb-dev.trantor.internal/health`
3. No `kubectl` or cluster-internal DNS in the playbook
4. `validate_certs: false` is set (self-signed `.trantor.internal` cert)
5. Retry logic: 5 retries, 15s delay
6. Playbook asserts `status == 'pass'` in response JSON
7. Playbook exits non-zero if health check fails
8. Playbook is non-destructive (read-only HTTP check)

## Tasks / Subtasks

- [ ] Task 1: Read an existing verify playbook for pattern reference
  - [ ] Read `playbooks/verify-inference-gateway-dev-deploy.yml` or similar
  - [ ] Confirm it uses external DNS + `ansible.builtin.uri` (not kubectl)
  - [ ] Note retry config

- [ ] Task 2: Write `verify-influxdb-dev-deploy.yml`
  - [ ] Health check: `GET http://influxdb-dev.trantor.internal/health`
  - [ ] `validate_certs: false`
  - [ ] Retry: 5 retries, 15s delay
  - [ ] Assert `status == 'pass'` in response JSON
  - [ ] NO kubectl, NO svc.cluster.local

- [ ] Task 3: Verify prerequisite — Story 1.5 complete
  - [ ] Confirm `influxdb-dev.trantor.internal` resolves and routes to the service

- [ ] Task 4: Test playbook
  - [ ] `ansible-playbook playbooks/verify-influxdb-dev-deploy.yml`
  - [ ] Confirm passes when InfluxDB dev is healthy

- [ ] Task 5: Commit
  - [ ] `git commit -m "feat(monitoring): add verify-influxdb-dev-deploy playbook"`
