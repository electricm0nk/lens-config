# Story 5.1: Blast-and-Repave Procedure — Full Infrastructure Teardown and Rebuild (AR-S6 MEDIUM, FR13, FR30, NFR18)

Status: ready-for-dev

## Story

As the operator,
I want a documented and validated blast-and-repave procedure (`tofu destroy && tofu apply`) that reproductions the entire Vault infrastructure from scratch using only the age key as human input,
so that the system can recover from catastrophic VM failure or compromise within the documented RTO.

## Acceptance Criteria

1. **[Single human input requirement (NFR18)]** Given a complete blast-and-repave is initiated, when I run `tofu destroy` followed by `tofu apply`, then the only human-supplied input required is the `SOPS_AGE_KEY_FILE` environment variable (pointing to the age private key file). All other parameters are derived from Terraform state, SOPS-encrypted files, or provider configuration.

2. **[Infrastructure reprovisioned to operational state]** Given `tofu apply` completes after `tofu destroy`, when I run `vault status`, then the response shows `Initialized: true` and `Sealed: false` — Vault is initialized, unsealed, and serving API requests.

3. **[All 3 AppRole identities reprovisioned]** Given the full repave is complete, when I run `vault list auth/approle/role`, then all three workload identities are present: `opentofu`, `k3s`, `openclaw`.

4. **[Isolation verification passes post-repave]** Given the repave is complete and all AppRole identities are reprovisioned, when I run `ansible-playbook ansible/playbooks/verify-isolation.yml`, then all checks pass and the output reads: `ISOLATION VERIFICATION: ALL CHECKS PASSED`.

5. **[Runbook contains Verified Execution Record section]** Given `docs/runbooks/blast-and-repave.md` is created, when I read it, then it contains a `## Verified Execution Record` section with the following table populated from a real execution:
   | Field | Value |
   |---|---|
   | Date | <ISO date of test execution> |
   | Operator | <operator name> |
   | Duration (wall clock) | <time from tofu destroy start to vault status green> |
   | Outcome | PASS / FAIL |
   | RTO vs Target | <actual time> vs <NFR13 target> |

6. **[Runbook contains "What Can Go Wrong" section]** Given the runbook, when I read it, then a `## What Can Go Wrong` section documents at minimum these 4 failure scenarios:
   - Age key unavailable (SOPS_AGE_KEY_FILE not set or file missing)
   - Proxmox API unreachable (provider cannot connect to Proxmox host)
   - Vault init output not captured (init keys printed but not saved to SOPS)
   - Vault unsealed but AppRole identities missing (vault-config module failed mid-apply)

## Tasks / Subtasks

- [ ] **Task 1: Create `docs/runbooks/blast-and-repave.md`** (AC: 5, 6)
  - [ ] Create the full runbook:
    ```markdown
    # Blast-and-Repave Runbook

    ## Overview
    Blast-and-repave is a full infrastructure destruction and rebuild using OpenTofu.
    Use when: VM is compromised, storage is corrupted, or infrastructure needs a clean rebuild.
    The procedure is fully automated given the age private key.

    ## Prerequisites

    - `SOPS_AGE_KEY_FILE` set and pointing to the age private key (the ONLY required human input, NFR18)
    - Proxmox API accessible
    - Off-VM Raft snapshots available on `snapshot_host` for reference (not required for repave — repave rebuilds from IaC, not from snapshot)

    ## Procedure

    ### Step 1: Confirm prerequisites
    ```sh
    echo $SOPS_AGE_KEY_FILE              # must be set
    ls -la $SOPS_AGE_KEY_FILE            # must exist and be readable
    vault operator raft snapshot list    # optional: record last snapshot timestamp
    ```

    ### Step 2: Destroy all infrastructure
    ```sh
    cd <root_module_dir>
    tofu destroy
    ```
    Review the destroy plan carefully. Confirm the plan includes VM, TLS certs, and Vault backend resources.
    Type `yes` when prompted.

    ### Step 3: Verify clean state
    ```sh
    tofu show    # should show empty state
    # Verify VM is gone in Proxmox console
    ```

    ### Step 4: Reprovision all infrastructure
    ```sh
    tofu apply
    ```
    Only input required: none (SOPS_AGE_KEY_FILE is already set in environment).
    The `null_resource` Vault init step (Story 1.6) will re-initialize Vault, generate new unseal keys,
    and store them in the SOPS bootstrap-creds file via the provisioner.

    ### Step 5: Confirm Vault is operational
    ```sh
    vault status
    # Expected: Initialized: true, Sealed: false
    ```

    ### Step 6: Verify AppRole identities
    ```sh
    vault list auth/approle/role
    # Expected: opentofu, k3s, openclaw
    ```

    ### Step 7: Run isolation verification
    ```sh
    ansible-playbook ansible/playbooks/verify-isolation.yml
    # Expected: ISOLATION VERIFICATION: ALL CHECKS PASSED
    ```

    ## Verified Execution Record

    | Field | Value |
    |---|---|
    | Date | TBD — populate after first execution |
    | Operator | TBD |
    | Duration (wall clock) | TBD |
    | Outcome | TBD |
    | RTO vs Target | TBD vs NFR13 target |

    ## What Can Go Wrong

    ### Age key unavailable
    **Symptom:** `tofu apply` fails with SOPS decryption error or provisioner cannot read bootstrap creds.
    **Resolution:** Restore age private key from offline custody (see `docs/runbooks/age-key-custody.md`).
    **Prevention:** Always verify SOPS_AGE_KEY_FILE before starting.

    ### Proxmox API unreachable
    **Symptom:** `tofu apply` fails at `proxmox_vm_qemu` resource with connection error.
    **Resolution:** Verify Proxmox host is accessible; check provider credentials in environment.
    **Prevention:** Test Proxmox API connectivity before repave: `curl -k https://<proxmox_ip>:8006/api2/json`.

    ### Vault init output not captured
    **Symptom:** `tofu apply` completes, Vault is initialized, but unseal keys and root token are unknown.
    **Resolution:** Run `vault operator init` manually (Vault will report "already initialized"); root token is not recoverable — break-glass procedure cannot be used. Consider destroying and reapplying.
    **Prevention:** Ensure the null_resource vault init provisioner writes to SOPS bootstrap creds. Test this in a non-production environment first.

    ### Vault operational but AppRole identities missing
    **Symptom:** `vault status` is green but `vault list auth/approle/role` returns empty.
    **Resolution:** The vault-config module apply failed. Run `tofu apply` again — it should be idempotent and apply only the remaining resources.
    **Recovery command:** `tofu apply -target module.vault_config`
    ```

- [ ] **Task 2: Execute blast-and-repave in a test run** (AC: 1, 2, 3, 4)
  - [ ] **IMPORTANT:** This is a destructive test. Only execute in a dedicated test/staging environment or when explicitly ready to rebuild production. Confirm go/no-go with the operator.
  - [ ] Set `SOPS_AGE_KEY_FILE` and confirm no other env vars required
  - [ ] Run `tofu destroy` → confirms plan, type `yes`
  - [ ] Run `tofu apply` → completes without additional prompts (AC: 1)
  - [ ] Verify: `vault status` → Initialized + Unsealed (AC: 2)
  - [ ] Verify: `vault list auth/approle/role` → opentofu, k3s, openclaw (AC: 3)
  - [ ] Verify: `ansible-playbook verify-isolation.yml` → all checks pass (AC: 4)

- [ ] **Task 3: Populate Verified Execution Record** (AC: 5)
  - [ ] Record start time (before `tofu destroy`) and end time (after `vault status` green)
  - [ ] Calculate wall-clock duration
  - [ ] Update the runbook `## Verified Execution Record` table with actual values
  - [ ] Compare actual duration against NFR13 RTO target — note pass or fail

## Dev Notes

### Architecture Constraints

- **AR-S6 — MEDIUM: Blast-and-repave not yet validated:** "Recovery from tofu destroy + tofu apply not yet tested end-to-end." This story validates that path and closes AR-S6 by producing the Verified Execution Record with an actual run.
  [Source: docs/terminus/infra/secrets/architecture.md#Disaster Recovery Decisions]

- **NFR18 — Single human input:** "Blast-and-repave requires only: SOPS_AGE_KEY_FILE." This means: no manual unseal key entry, no manual root token copy-paste, no interactive Vault init prompts. The `null_resource` init provisioner (Story 1.6) and SOPS automation (Story 2.2) must handle all credential setup. If additional human inputs are required, NFR18 is violated and the earlier stories must be revisited.

- **FR13 — Complete infrastructure rebuild:** The blast-and-repave covers the FULL stack: VM + TLS + Vault installation + AppRole identities. It does NOT restore Vault KV secrets from Raft snapshot (that is Story 5.2, snapshot restore). A blast-and-repave produces a clean Vault instance; secrets must be re-synced from SOPS files (by running `tofu apply` on vault-config which includes the `data "sops_file"` → `vault_kv_secret_v2` resource pipeline from Story 2.4).

- **FR30 — Tested in non-production:** "Blast-and-repave MUST be tested before production use." Story 5.1 validates this by executing the procedure and recording results. Do NOT mark this story complete without actually running the procedure.

- **Depends on Stories 1.6, 2.2, 2.4:** The blast-and-repave depends on: null_resource init provisioner (1.6), SOPS bootstrap creds (2.2), SOPS-to-Vault sync (2.4). All must be valid for NFR18 to hold. If any of those stories have gaps, they must be fixed before this story can be completed.

### Project Structure Notes

- Creates `docs/runbooks/blast-and-repave.md` (new file)
- No changes to infrastructure code

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 5.1]
- [Source: docs/terminus/infra/secrets/architecture.md#Disaster Recovery — AR-S6/FR13/FR30/NFR18]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
