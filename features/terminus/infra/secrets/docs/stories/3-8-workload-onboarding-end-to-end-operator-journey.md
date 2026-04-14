# Story 3.8: Workload Onboarding End-to-End — Operator Journey Validation (AR-S7 MEDIUM, FR6, FR14)

Status: ready-for-dev

## Story

As the operator,
I want a complete workload onboarding runbook validated by a test-workload end-to-end exercise,
so that adding a new AppRole identity to the system follows a repeatable, documented process that leaves no trace after the test workload is removed.

## Acceptance Criteria

1. **[Runbook exists with 6-step checklist]** Given `docs/runbooks/workload-onboarding.md` is created, when I read it, then it contains a `## Onboarding Checklist` section with exactly these 6 steps:
   1. Create policy HCL in `vault-config/policies/<workload>.hcl`
   2. Add `vault_policy` and `vault_approle_auth_backend_role` resources to vault-config module
   3. Add `approle_role_id_<workload>` output to `vault-config/outputs.tf`
   4. Add isolation test for the new workload to `ansible/playbooks/verify-isolation.yml`
   5. Run `tofu apply` on vault-config module
   6. Run `ansible-playbook ansible/playbooks/verify-isolation.yml` — confirm all checks pass

2. **[test-workload onboarded successfully]** Given the 6-step checklist is followed for a `test-workload` identity, when `tofu apply` completes, then `vault policy read test-workload` returns the test-workload policy and `vault read auth/approle/role/test-workload` returns the role configuration.

3. **[test-workload secret accessible after onboarding]** Given the test-workload AppRole is provisioned and a test secret seeded at `secret/terminus/test/test-workload/verification_key`, when I log in with the test-workload AppRole and attempt to read that secret, then the read succeeds.

4. **[verify-isolation.yml passes with test-workload included]** Given the test-workload check is added to `verify-isolation.yml` (per checklist step 4), when the playbook runs, then all checks pass including the new test-workload check.

5. **[test-workload cleanly removed]** Given the test-workload was onboarded (AC2, AC3), when I remove the test-workload resources from the vault-config module and run `tofu apply`, then `vault policy read test-workload` returns "not found" (policy removed), `vault read auth/approle/role/test-workload` returns "not found" (role removed), and `tofu plan` shows no pending changes.

6. **[Post-removal verify-isolation.yml still passes]** Given the test-workload is removed and its check removed from `verify-isolation.yml`, when I run the playbook, then the remaining checks (opentofu, openclaw, k3s) all still pass — removal of one workload does not affect others.

## Tasks / Subtasks

- [ ] **Task 1: Create `docs/runbooks/workload-onboarding.md`** (AC: 1)
  - [ ] Create the runbook with sections:
    - `# Vault Workload Onboarding Runbook`
    - `## Overview` — brief description of AppRole-based workload identity model
    - `## Onboarding Checklist` — the 6-step numbered list (per AC1)
    - `## Token TTL Policy` — from Story 3.2 (TTL table: opentofu, k3s, openclaw)
    - `## Secret-ID Rotation and Revocation` — from Story 3.6 (accessor commands, wrapped token delivery)
    - `## Off-boarding a Workload` — how to remove a workload identity (reverse of onboarding checklist)
    - `## Policy HCL Template` — copy/paste-ready HCL stub for new workloads:
      ```hcl
      # Policy for <workload> AppRole — <description>

      path "secret/data/terminus/<namespace>/<workload>/*" {
        capabilities = ["read", "list"]
      }

      path "secret/metadata/terminus/<namespace>/<workload>/*" {
        capabilities = ["read", "list"]
      }

      path "sys/leases/renew" {
        capabilities = ["update"]
      }
      ```
    - `## Verified Onboarding Records` — table of workloads onboarded with date:
      | Workload | Namespace | Onboarded | Policy Path |
      |---|---|---|---|
      | opentofu | infra | <date> | vault-config/policies/opentofu.hcl |
      | k3s | infra/k3s | <date> | vault-config/policies/k3s.hcl |
      | openclaw | agent/openclaw | <date> | vault-config/policies/openclaw.hcl |

- [ ] **Task 2: Onboard test-workload following the checklist** (AC: 2)
  - [ ] Create `modules/vault-config/policies/test-workload.hcl`:
    ```hcl
    # Policy for test-workload AppRole — read-only verification identity

    path "secret/data/terminus/test/test-workload/*" {
      capabilities = ["read", "list"]
    }

    path "secret/metadata/terminus/test/test-workload/*" {
      capabilities = ["read", "list"]
    }

    path "sys/leases/renew" {
      capabilities = ["update"]
    }
    ```
  - [ ] Add to `modules/vault-config/main.tf`:
    ```hcl
    resource "vault_policy" "test_workload" {
      name   = "test-workload"
      policy = file("${path.module}/policies/test-workload.hcl")
    }

    resource "vault_approle_auth_backend_role" "test_workload" {
      backend        = "approle"
      role_name      = "test-workload"
      token_policies = [vault_policy.test_workload.name]
      token_ttl      = 3600
      token_max_ttl  = 86400
      secret_id_num_uses = 0
    }
    ```
  - [ ] Add to `modules/vault-config/outputs.tf`:
    ```hcl
    output "approle_role_id_test_workload" {
      description = "Test workload AppRole role_id — remove after onboarding validation"
      value       = vault_approle_auth_backend_role.test_workload.role_id
      sensitive   = false
    }
    ```
  - [ ] Run: `tofu apply` on vault-config module → confirm creates 2 resources + 1 output
  - [ ] Verify: `vault policy read test-workload` and `vault read auth/approle/role/test-workload` both return data (AC: 2)

- [ ] **Task 3: Seed test secret and verify access** (AC: 3)
  - [ ] Seed: `vault kv put secret/terminus/test/test-workload/verification_key value="e2e-test-$(date +%s)"`
  - [ ] Generate secret-id: `vault write -f auth/approle/role/test-workload/secret-id`
  - [ ] Login: `vault write auth/approle/login role_id=$(tofu output approle_role_id_test_workload) secret_id=<generated>`
  - [ ] Verify read: `vault kv get secret/terminus/test/test-workload/verification_key` → returns value (AC: 3)

- [ ] **Task 4: Add test-workload check to verify-isolation.yml and run** (AC: 4)
  - [ ] Add a 4th check to `ansible/playbooks/verify-isolation.yml`:
    - Check 4: test-workload reads `secret/terminus/test/test-workload/verification_key` → PASS
    - Check 5 (optional): test-workload cannot read `secret/terminus/infra/postgres/app_role_password` → PASS (403)
  - [ ] Run `ansible-playbook ansible/playbooks/verify-isolation.yml` → confirm all checks including test-workload pass (AC: 4)

- [ ] **Task 5: Remove test-workload and verify clean removal** (AC: 5)
  - [ ] Remove from `modules/vault-config/main.tf`: delete `vault_policy.test_workload` and `vault_approle_auth_backend_role.test_workload` blocks
  - [ ] Remove from `modules/vault-config/outputs.tf`: delete `approle_role_id_test_workload` output
  - [ ] Delete `modules/vault-config/policies/test-workload.hcl`
  - [ ] Run: `tofu plan` → confirm shows 2 destroys (policy + role)
  - [ ] Run: `tofu apply` → confirm destroys succeed
  - [ ] Verify: `vault policy read test-workload` → "no policy named test-workload" (AC: 5)
  - [ ] Verify: `vault read auth/approle/role/test-workload` → "no role" response (AC: 5)
  - [ ] Run: `tofu plan` → confirms no pending changes (AC: 5)

- [ ] **Task 6: Remove test-workload check from verify-isolation.yml and re-run** (AC: 6)
  - [ ] Remove the test-workload tasks from `ansible/playbooks/verify-isolation.yml`
  - [ ] Run: `ansible-playbook ansible/playbooks/verify-isolation.yml` → confirm "ISOLATION VERIFICATION: ALL CHECKS PASSED" with original 3 checks (AC: 6)

- [ ] **Task 7: Add test-workload exercise record to runbook** (AC: 1)
  - [ ] Update `docs/runbooks/workload-onboarding.md` Verified Onboarding Records table with test-workload entry, marking it as "removed after validation"
  - [ ] Add a final note: "test-workload onboarded and cleanly removed on <date> — E2E validation complete"

## Dev Notes

### Architecture Constraints

- **AR-S7 — MEDIUM: No onboarding runbook for new AppRole identities:** "A runbook for onboarding new workload identities (AppRole + policy + isolation test) must be documented." This story directly closes AR-S7 by producing `workload-onboarding.md` and validating it end-to-end with the test-workload exercise.
  [Source: docs/terminus/infra/secrets/architecture.md#AppRole Vault Integration — AR-S7]

- **FR14 — Documented operational procedures:** The workload-onboarding.md is a required operational procedure document per FR14. It must be production-ready (not a draft) by the time this story is complete.

- **test-workload is ephemeral:** The test-workload resources added in Tasks 2–4 are explicitly temporary. They exercise the onboarding process but should not be committed as long-lived infrastructure. Ensure the final commit for this story has them removed (Task 5 must be complete before commit).

- **Policy file cleanup:** `modules/vault-config/policies/test-workload.hcl` must also be deleted before commit (deleted in Task 5 step 3).

- **Commit sequencing:** The correct commit sequence for this story is:
  1. Commit: runbook + test-workload resources (onboarding exercise)
  2. Commit: remove test-workload resources (cleanup)
  OR
  1. Single commit: runbook only (after full exercise is complete and test-workload removed)
  The second approach is cleaner for the PR review — exercise, then commit only the permanent artifacts.

### Project Structure Notes

- Creates `docs/runbooks/workload-onboarding.md` (permanent artifact)
- Temporarily creates and then removes `modules/vault-config/policies/test-workload.hcl` (ephemeral)
- Temporarily adds and then removes test-workload resources in `modules/vault-config/main.tf` and `outputs.tf`

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 3.8]
- [Source: docs/terminus/infra/secrets/architecture.md#AppRole Vault Integration — AR-S7]
- [Source: docs/terminus/infra/secrets/architecture.md#Additional Requirements — FR14]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
