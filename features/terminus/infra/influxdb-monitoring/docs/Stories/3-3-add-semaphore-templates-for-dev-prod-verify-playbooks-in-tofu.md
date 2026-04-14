# Story 3.3: Add Semaphore Templates for Dev/Prod Verify Playbooks in Tofu

Status: done

**Epic:** 3 — Verification and Release Integration
**Initiative:** terminus-infra-influxdb-monitoring
**Date:** 2026-04-12
**Depends on:** Stories 3.1, 3.2
**Implemented:** this session

---

## Story

As an infrastructure operator,
I want Semaphore task templates for the InfluxDB dev and prod verify playbooks,
so that verification can be triggered on-demand or scheduled via Semaphore CI without manual `ansible-playbook` invocation.

---

## Context

**Repo:** `terminus.infra`
**File:** `tofu/environments/semaphore/main.tf`
**Pattern:** Follows existing Semaphore template resources in main.tf
**Provisioner:** OpenTofu 1.11.0

**Semaphore project assignment — CRITICAL (cicd.md):**
There is ONE Semaphore project managed in tofu: `semaphoreui_project.terminus_infra`. ALL templates — bootstrap ops, release templates, and verify templates — live in this single project resource. The numeric IDs assigned by Semaphore (project 2 = `terminus-infra-ops`, project 3 = `terminus-infra`) refer to creation order, not separate tofu resources.

**Correct resource type:** `semaphoreui_project_template` (not `semaphore_template`)

---

## Technical Notes

**New terraform resources added (in `tofu/environments/semaphore/main.tf`):**

```hcl
resource "semaphoreui_project_template" "verify_influxdb_dev_deploy" {
  project_id     = semaphoreui_project.terminus_infra.id
  name           = "verify-influxdb-dev-deploy"
  playbook       = "platforms/k3s/ansible/playbooks/verify-influxdb-dev-deploy.yml"
  inventory_id   = semaphoreui_project_inventory.k3s.id
  repository_id  = semaphoreui_project_repository.terminus_infra.id
  environment_id = semaphoreui_project_environment.default.id
  description    = "Smoke test InfluxDB dev deployment — /health 200 and status pass"
}

resource "semaphoreui_project_template" "verify_influxdb_deploy" {
  project_id     = semaphoreui_project.terminus_infra.id
  name           = "verify-influxdb-deploy"
  playbook       = "platforms/k3s/ansible/playbooks/verify-influxdb-deploy.yml"
  inventory_id   = semaphoreui_project_inventory.k3s.id
  repository_id  = semaphoreui_project_repository.terminus_infra.id
  environment_id = semaphoreui_project_environment.default.id
  description    = "Smoke test InfluxDB deployment — /health 200 and status pass"
}
```

---

## Acceptance Criteria

1. Two new `semaphoreui_project_template` resources added to `tofu/environments/semaphore/main.tf`
2. Both templates reference `semaphoreui_project.terminus_infra.id` (the single managed project)
3. Dev template: `name = "verify-influxdb-dev-deploy"`, playbook path includes `platforms/k3s/ansible/playbooks/`
4. Prod template: `name = "verify-influxdb-deploy"`, same path prefix
5. `tofu plan` shows only 2 new template resources
6. `tofu apply` succeeds — templates visible in Semaphore UI

## Tasks / Subtasks

- [ ] Task 1: Read existing `tofu/environments/semaphore/main.tf`
  - [ ] Identify project 2 resource name (`semaphore_project.terminus_infra_ops` or equivalent)
  - [ ] Confirm project 2 is `terminus-infra-ops` (manual cluster ops)
  - [ ] Identify data source names for inventory, repository, environment
  - [ ] Note `type` value and any survey_vars used by other infra ops templates

- [ ] Task 2: Add `semaphore_template.verify_influxdb_dev` resource
  - [ ] Match naming convention of existing templates
  - [ ] Set `playbook = "playbooks/verify-influxdb-dev-deploy.yml"`

- [ ] Task 3: Add `semaphore_template.verify_influxdb_prod` resource
  - [ ] Set `playbook = "playbooks/verify-influxdb-deploy.yml"`

- [ ] Task 4: Run tofu plan
  - [ ] `cd tofu/environments/semaphore && tofu plan`
  - [ ] Verify plan shows only 2 new resources

- [ ] Task 5: Apply and verify
  - [ ] `tofu apply`
  - [ ] Confirm both templates appear in Semaphore UI under the terminus project

- [ ] Task 6: Commit
  - [ ] `git commit -m "feat(semaphore): add influxdb dev/prod verify templates to tofu"`
