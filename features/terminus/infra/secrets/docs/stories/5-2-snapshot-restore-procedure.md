# Story 5.2: Snapshot Restore Procedure — Raft Snapshot Recovery Validation (FR29, NFR19, NFR13)

Status: ready-for-dev

## Story

As the operator,
I want a documented and validated snapshot restore procedure using `vault operator raft snapshot restore`,
so that Vault's KV secrets and auth configuration can be recovered from a Raft snapshot within the RTO target.

## Acceptance Criteria

1. **[Restore command executes successfully]** Given a valid Raft snapshot file exists on the operator machine (sourced from Story 4.3 automated snapshots or Story 4.6 on-demand snapshot), when I run `vault operator raft snapshot restore <snapshot_file>`, then the command exits 0 and Vault reports `Initialized: true, Sealed: false` after the restore.

2. **[Three-part acceptance test passes after restore (FR19)]**  Given the snapshot restore is complete, when I run the three-part verification:
   - Part 1: `vault kv get secret/terminus/infra/postgres/app_role_password` → returns data (secrets readable)
   - Part 2: `vault write auth/approle/login role_id=<opentofu_role_id> secret_id=<valid_secret_id>` → returns a token (AppRole auth functional)
   - Part 3: `tail -10 /var/log/vault/audit.log` → shows new entries within 60s of operations above (audit log active)
   then all three parts succeed.

3. **[RTO ≤ 4 hours (NFR13)]** Given the snapshot restore procedure, when executed from start to finish (snapshot retrieval from off-VM host + restore command + three-part acceptance test), then the total elapsed wall-clock time is ≤ 4 hours. The measured RTO is recorded in `docs/runbooks/snapshot-restore.md`.

4. **[Runbook contains RTO measurement]** Given `docs/runbooks/snapshot-restore.md` is created, when I read it, then the `## Measured RTO` section contains the actual wall-clock duration from a validated execution.

5. **[Decision guide: restore vs repave]** Given the runbook, when I read the `## Decision Guide` section, then it provides clear criteria for choosing between snapshot restore (Story 5.2) and blast-and-repave (Story 5.1):
   - Use snapshot restore when: VM is intact, Vault data is corrupt or accidentally deleted, known-good snapshot is recent
   - Use blast-and-repave when: VM is compromised or destroyed, snapshot is unavailable or untrusted, clean rebuild is preferred

6. **[Restore requires unseal after restore]** Given the runbook procedure, when I read the steps, then Step N explicitly states: "After snapshot restore, Vault will be sealed. Unseal using the unseal key from `sops/bootstrapcreds.sops.yaml`." This is documented because snapshot restore does not restore the unseal state.

## Tasks / Subtasks

- [ ] **Task 1: Retrieve a snapshot for testing** (AC: 1, 2)
  - [ ] From `snapshot_host`, copy a recent daily snapshot to the operator machine:
    `rsync <snapshot_host>:/vault-snapshots/daily/<latest_snapshot>.snap /tmp/test-restore.snap`
  - [ ] Verify: `ls -la /tmp/test-restore.snap` — confirms non-zero size

- [ ] **Task 2: Execute snapshot restore** (AC: 1, 6)
  - [ ] **CAUTION:** Snapshot restore replaces ALL Vault data with the snapshot contents. Ensure this is a planned recovery exercise, not an accidental invocation.
  - [ ] Vault must be initialized and unsealed before restore (restore operation requires an authenticated token):
    ```sh
    export VAULT_TOKEN=<root_or_ops_token>
    vault operator raft snapshot restore /tmp/test-restore.snap
    ```
  - [ ] After restore, Vault will be sealed — unseal it:
    ```sh
    sops -d sops/bootstrapcreds.sops.yaml | grep unseal_key  # get unseal key
    vault operator unseal <unseal_key>
    ```
  - [ ] Verify: `vault status` → Initialized: true, Sealed: false (AC: 1)

- [ ] **Task 3: Run three-part acceptance test** (AC: 2)
  - [ ] Part 1 — Secrets readable:
    `vault kv get secret/terminus/infra/postgres/app_role_password` → returns data
  - [ ] Part 2 — AppRole auth functional:
    `vault write auth/approle/login role_id=<opentofu_role_id> secret_id=<valid_secret_id>` → returns token
  - [ ] Part 3 — Audit log active:
    `tail -10 /var/log/vault/audit.log` → shows timestamped entries within 60s of the operations above
  - [ ] Record pass/fail for each part

- [ ] **Task 4: Time the RTO and record it** (AC: 3, 4)
  - [ ] Record start time: when rsync of snapshot began
  - [ ] Record end time: when Part 3 of acceptance test passed
  - [ ] Calculate: `end - start` in minutes/hours
  - [ ] Confirm ≤ 4 hours (NFR13)

- [ ] **Task 5: Create `docs/runbooks/snapshot-restore.md`** (AC: 4, 5, 6)
  - [ ] Create the runbook with sections:
    ```markdown
    # Snapshot Restore Runbook

    ## Overview
    Vault Raft snapshot restore recovers Vault's secrets and configuration from a previously
    taken snapshot. Use when Vault data is accidentally deleted or corrupted but the VM is intact.

    ## Decision Guide

    | Situation | Use |
    |---|---|
    | VM intact, data corrupt/deleted | Snapshot restore (this runbook) |
    | VM compromised or destroyed | Blast-and-repave (docs/runbooks/blast-and-repave.md) |
    | Snapshot unavailable or untrusted | Blast-and-repave + re-sync from SOPS |
    | Known-good snapshot available | Snapshot restore (faster recovery) |

    ## Prerequisites

    - Valid Raft snapshot on `snapshot_host` (from daily cron, Story 4.3, or on-demand, Story 4.6)
    - Vault running (initialized, either sealed or unsealed)
    - `VAULT_TOKEN` with root or operator-level permissions
    - Age private key (`SOPS_AGE_KEY_FILE`) to decrypt bootstrap creds for unseal key

    ## Procedure

    ### Step 1: Retrieve snapshot from off-VM host
    ```sh
    # List available snapshots
    ssh <snapshot_host> ls -lt /vault-snapshots/daily/ | head -5
    # Copy the desired snapshot
    rsync <snapshot_host>:/vault-snapshots/daily/<filename>.snap /tmp/restore.snap
    ```

    ### Step 2: Verify snapshot integrity
    ```sh
    ls -la /tmp/restore.snap    # must be non-zero size
    # Optional: validate snapshot is a valid Raft snapshot
    vault operator raft snapshot inspect /tmp/restore.snap
    ```

    ### Step 3: Restore snapshot
    ```sh
    export VAULT_ADDR=https://<vault_ip>:8200
    export VAULT_CACERT=<path_to_ca.pem>
    export VAULT_TOKEN=<root_or_ops_token>
    vault operator raft snapshot restore /tmp/restore.snap
    ```

    ### Step 4: Unseal Vault (restore ALWAYS seals Vault)
    ```sh
    UNSEAL_KEY=$(sops -d sops/bootstrapcreds.sops.yaml | grep unseal_key | awk '{print $2}')
    vault operator unseal $UNSEAL_KEY
    vault status    # Confirm: Initialized: true, Sealed: false
    ```

    ### Step 5: Run acceptance test

    #### Part 1 — Secrets readable
    ```sh
    vault kv get secret/terminus/infra/postgres/app_role_password
    # Expected: returns data with current_version
    ```

    #### Part 2 — AppRole auth functional
    ```sh
    vault write auth/approle/login \
      role_id=$(tofu output approle_role_id_opentofu) \
      secret_id=<valid_secret_id>
    # Expected: returns client_token
    ```

    #### Part 3 — Audit log active
    ```sh
    tail -10 /var/log/vault/audit.log
    # Expected: JSON entries within 60s of operations above
    ```

    ## Measured RTO

    | Execution | Date | Duration | Outcome |
    |---|---|---|---|
    | Initial validation | TBD | TBD | TBD |

    NFR13 target: ≤ 4 hours from snapshot retrieval to verified restore.

    ## Important Notes

    - **Snapshot restore replaces ALL Vault data** with the snapshot contents. Any changes made
      after the snapshot was taken are lost. Review snapshot timestamp before restoring.
    - **AppRole secret-ids generated after the snapshot will not be present** in the restored state.
      Re-generate secret-ids for all workloads after restore.
    - **Vault audit log is NOT part of the Raft snapshot.** The audit log at `/var/log/vault/audit.log`
      is a filesystem file, not stored in Raft. The audit log from before the restore remains intact.
    ```

## Dev Notes

### Architecture Constraints

- **FR29 — Validated snapshot recovery:** "Snapshot restore procedure verified end-to-end and documented." This story explicitly validates the procedure (not just documents it) by running a live restore and recording the result.

- **NFR19 — Documented snapshot restore procedure:** The runbook is the compliance artifact. It must contain the actual RTO measurement from execution (not a placeholder).

- **NFR13 — RTO ≤ 4 hours:** "Vault recoverable from snapshot within 4-hour RTO target." This applies to snapshot restore. The blast-and-repave RTO is measured separately in Story 5.1. Measure and record actual restore duration in the runbook.

- **Snapshot restore seals Vault:** This is a commonly missed operational detail. `vault operator raft snapshot restore` ALWAYS seals Vault after the restore. The operator MUST unseal using the unseal key. The runbook MUST document this as Step 4. If the unseal key is in SOPS bootstrap creds, it is recoverable via `sops -d sops/bootstrapcreds.sops.yaml`.

- **AppRole secret-id post-restore:** Secret-ids generated after the snapshot was taken will not exist in the restored state (they are part of Vault's Raft data). All workloads (opentofu, k3s, openclaw) will need new secret-ids generated after restore. Document this in runbook Important Notes.

- **Audit log is NOT in Raft snapshot:** The Vault audit log is a filesystem file at `/var/log/vault/audit.log`. It is NOT part of the Raft state machine. After snapshot restore, the audit log continues from where it was before the restore. This means audit log entries from AFTER the snapshot (but before the restore) remain in the log.

### Project Structure Notes

- Creates `docs/runbooks/snapshot-restore.md` (new file, replaces stub from Story 4.6)
- No changes to infrastructure code

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 5.2]
- [Source: docs/terminus/infra/secrets/architecture.md#Disaster Recovery — FR29/NFR19/NFR13]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
