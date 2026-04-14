# Story 5.3: Break-Glass Procedure — Emergency Root Access with Mandatory Audit Logging (AR-S5 MEDIUM, FR31, FR32, NFR14)

Status: ready-for-dev

## Story

As the operator,
I want a documented break-glass procedure for emergency Vault root access that mandates an ops-log.md entry and ends with root token revocation,
so that emergency access is possible when all other auth mechanisms fail, yet every such session is auditable and time-bounded.

## Acceptance Criteria

1. **[Runbook exists with required structure]** Given `docs/runbooks/break-glass.md` is created, when I read it, then it contains all of these sections: `## Prerequisites`, `## Procedure`, `## Mandatory Ops-Log Entry`, `## Post-Session Validation`, and `## Important Notes`.

2. **[Procedure requires unseal key custody retrieval]** Given the `## Procedure` section, when I read Step 1, then it documents how to retrieve the unseal key from `sops/bootstrapcreds.sops.yaml` (decrypted with SOPS_AGE_KEY_FILE), which is the offline unseal key custody mechanism established in Story 2.2.

3. **[Root token revocation is the FINAL command (FR32)]** Given the `## Procedure`, when I read the last step, then it states that `vault token revoke <root_token>` (or equivalent) is the FINAL command executed — no operations may be performed after root token revocation. The runbook must include a warning: "Skipping revocation is a security violation."

4. **[ops-log.md entry is mandatory before root token revocation (FR31)]** Given the `## Mandatory Ops-Log Entry`, when I read it, then it states: "You MUST create a `docs/ops-log.md` entry BEFORE revoking the root token. The entry format requires: timestamp, reason, operations performed, and revocation confirmation." The runbook must make clear that revocation without an ops-log entry violates FR31.

5. **[ops-log.md contains at least one real entry]** Given `docs/ops-log.md` exists (created in Story 4.6 or here), when I read it, then it contains at least one completed entry in the required format — either from an actual break-glass session or from a validation drill.

6. **[Cross-reference between ops-log and audit log (FR32)]** Given the ops-log entry format, when I read the entry template, then it includes a field: `Vault Audit Reference: <ISO timestamp range that covers this session>`. This field allows cross-referencing the ops-log entry with the corresponding Vault audit log entries.

7. **[AR-S5 risk is referenced and acknowledged]** Given the runbook, when I read it, then it references `AR-S5` and acknowledges: "Break-glass access uses a single-operator 1-of-1 Shamir unseal key (deviation from dual-control). Compensating control: mandatory ops-log entry and Vault audit log capture."

## Tasks / Subtasks

- [ ] **Task 1: Create `docs/runbooks/break-glass.md`** (AC: 1, 2, 3, 4, 6, 7)
  - [ ] Create the runbook:

    ```markdown
    # Break-Glass Procedure

    > **AR-S5 (MEDIUM):** Break-glass access uses a single-operator 1-of-1 Shamir unseal key,
    > which deviates from dual-control best practices. Compensating controls: mandatory
    > `docs/ops-log.md` entry and Vault audit log capture of all operations.

    ## Overview
    Break-glass is emergency Vault root access when all other authentication mechanisms fail
    (e.g., all AppRole secret-ids are lost, token store is inaccessible, Vault is sealed and
    the unseal key holder is unavailable). Use ONLY when no alternative auth method is viable.

    ## Prerequisites

    - `SOPS_AGE_KEY_FILE` set and pointing to the age private key
    - Access to `sops/bootstrapcreds.sops.yaml` in the repository
    - `docs/ops-log.md` entry started (even if not completed — complete BEFORE revocation)

    ## Procedure

    ### Step 1: Retrieve unseal key from SOPS bootstrap credentials
    ```sh
    sops -d sops/bootstrapcreds.sops.yaml
    # Retrieve the `vault_unseal_key` value
    UNSEAL_KEY="<unseal_key_value>"
    ```
    The unseal key is the 1-of-1 Shamir key generated during Story 1.6 Vault initialization.
    Offline custody: age-encrypted in `sops/bootstrapcreds.sops.yaml`.

    ### Step 2: Unseal Vault (if sealed)
    ```sh
    export VAULT_ADDR=https://<vault_ip>:8200
    export VAULT_CACERT=<path_to_ca.pem>
    vault operator unseal $UNSEAL_KEY
    vault status    # Confirm: Initialized: true, Sealed: false
    ```

    ### Step 3: Generate a new root token
    ```sh
    # Option A: If you have the unseal key (preferred for break-glass):
    vault operator generate-root -init
    # Complete OTP-based root token generation — follow Vault prompts
    # Record the generated root token in a secure temp location
    ```
    OR
    ```sh
    # Option B: Direct root token from bootstrap creds (only if stored there):
    ROOT_TOKEN=$(sops -d sops/bootstrapcreds.sops.yaml | grep vault_root_token | awk '{print $2}')
    # WARNING: Root token from Story 1.6 should have been revoked after bootstrap.
    # Option B is only valid if root token was NOT revoked (non-standard configuration).
    ```

    ### Step 4: Authenticate and perform required operations
    ```sh
    export VAULT_TOKEN=<root_token>
    vault token lookup    # Confirm root token is valid
    # Perform ONLY the operations required for the emergency:
    #   - Re-generate AppRole secret-ids if all are lost
    #   - Recover specific secrets if needed
    #   - Repair policy or auth configuration
    # Each operation will appear in the Vault audit log.
    ```

    ### Step 5: Create ops-log entry BEFORE revoking root token
    Open `docs/ops-log.md` and add an entry using the format in ## Mandatory Ops-Log Entry.
    Record the Vault audit log timestamp range covering this session.
    **You MUST complete this step before Step 6.**

    ### Step 6: Revoke the root token — FINAL COMMAND
    ```sh
    vault token revoke $VAULT_TOKEN
    # OR:
    # vault token revoke -self
    ```
    > ⚠ **WARNING:** This is the FINAL command of every break-glass session.
    > Skipping or deferring root token revocation is a security violation (FR32).
    > After this command, root access is no longer possible without repeating Steps 1–3.

    ## Mandatory Ops-Log Entry

    Each break-glass session requires an entry in `docs/ops-log.md` BEFORE root token revocation.
    The entry must use this format:

    ```markdown
    ## Break-Glass Session — <ISO Timestamp>
    - **Operator:** <name>
    - **Reason:** <brief justification for break-glass access>
    - **Operations Performed:**
      - [ ] <operation 1 — e.g., "Re-generated secret-id for k3s AppRole">
      - [ ] <operation 2>
    - **Revocation:** Root token revoked at <ISO timestamp of revocation>
    - **Vault Audit Reference:** <ISO timestamp range, e.g., 2026-03-24T02:15:00Z to 2026-03-24T02:28:00Z>
    - **Cross-Reference:** FR31 compliance — ops-log entry created before revocation ✅
    ```

    ## Post-Session Validation

    After break-glass session and root token revocation:
    1. Confirm Vault is healthy: `vault status` with non-root token
    2. Confirm required workload auth is restored: `vault list auth/approle/role`
    3. Confirm audit log captured the session: `grep "root" /var/log/vault/audit.log | tail -10`
    4. Commit ops-log.md entry to the repository: `git add docs/ops-log.md && git commit -m "ops(break-glass): session <date>"`

    ## Important Notes

    - **Dual-control deviation (AR-S5):** This system uses 1-of-1 Shamir (single unseal key).
      Dual-control would require a quorum (e.g., 3-of-5) of key holders to unseal. The ops-log
      entry and audit trail are the compensating controls for this deviation.
    - **Root token must always be revoked:** A persistent root token is a critical security risk.
      The original bootstrap root token was revoked in Story 1.6. Any break-glass root token
      must be revoked at the end of every session.
    - **ops-log entries are immutable once committed:** Do not amend or delete ops-log entries.
      They are the audit trail for emergency access sessions.
    - **SOPS_AGE_KEY_FILE is the master key:** Loss of the age private key means the bootstrap
      creds are inaccessible and break-glass may not be possible. See `docs/runbooks/age-key-custody.md`.
    ```

- [ ] **Task 2: Perform a break-glass validation drill** (AC: 5, 6)
  - [ ] Execute a break-glass drill in a non-critical maintenance window:
    - Retrieve unseal key from SOPS (Step 1)
    - Generate a temporary root token (Step 3)
    - Perform one benign operation (e.g., `vault token lookup`)
    - Create ops-log entry before revocation (Step 5)
    - Revoke root token (Step 6)
  - [ ] Verify: ops-log entry is present in `docs/ops-log.md` (AC: 5)
  - [ ] Verify: Vault audit log shows the session operations (cross-reference to ops-log entry)
  - [ ] Commit ops-log entry: `git add docs/ops-log.md && git commit -m "ops(break-glass): validation drill <date>"`

- [ ] **Task 3: Verify ops-log.md format and content** (AC: 5, 6)
  - [ ] Read `docs/ops-log.md`
  - [ ] Confirm at least one completed entry with all required fields (operator, reason, operations, revocation, audit reference)
  - [ ] Confirm the `Vault Audit Reference` timestamp range corresponds to actual audit log entries

## Dev Notes

### Architecture Constraints

- **AR-S5 — MEDIUM: 1-of-1 Shamir vs dual-control:** The architecture intentionally chose 1-of-1 Shamir (single unseal key, D3) for homelab simplicity. AR-S5 acknowledges this as a deviation from dual-control (e.g., NIST SP 800-57 key ceremony with multiple custodians). Compensating controls: mandatory ops-log entry + Vault audit log captures all operations. The runbook must reference this deviation explicitly.
  [Source: docs/terminus/infra/secrets/architecture.md#Vault Provisioning Decisions — D3/AR-S5]

- **FR31 — Break-glass sessions must be ops-logged:** Every break-glass session is a "privileged access" event that must generate an auditable record. The ops-log entry is the human-readable record; the Vault audit log is the cryptographic record. Both are required.

- **FR32 — Root token revocation is mandatory:** A persistent root Vault token is equivalent to complete system compromise. Root tokens must be revoked at the end of every session. The original root token from Story 1.6 initialization was revoked as part of that story. Any break-glass root token must follow the same pattern.

- **`vault operator generate-root` vs stored root token:** Best practice is to NOT store a root token anywhere (including SOPS). Instead, use `vault operator generate-root` to generate a fresh root token for each emergency session. The OTP-based generation requires the unseal key — which is in SOPS bootstrap creds. This is more secure than storing a persistent root token.

- **Ops-log immutability:** The `ops-log.md` file should be append-only. Use `git blame`/`git log` to establish an audit trail. Once an ops-log entry is committed, it must not be amended or removed — only corrections via a new entry.

### Project Structure Notes

- Creates `docs/runbooks/break-glass.md` (new file, replaces stub from Story 4.6)
- Modifies `docs/ops-log.md` (adds at least one completed entry)

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 5.3]
- [Source: docs/terminus/infra/secrets/architecture.md#Vault Provisioning Decisions — AR-S5/D3]
- [Source: docs/terminus/infra/secrets/architecture.md#Additional Requirements — FR31/FR32]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
