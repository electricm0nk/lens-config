# Story 4.3: Vault Raft Snapshot Automation — Daily Off-VM Backup via Ansible (D5, FR28, NFR10)

Status: ready-for-dev

## Story

As the operator,
I want a daily automated Vault Raft snapshot job deployed via Ansible that saves and rsync's snapshots to an off-VM storage host,
so that Vault state can be recovered in a disaster without relying on the Vault VM itself.

## Acceptance Criteria

1. **[Snapshot role exists and is applied]** Given the `ansible/roles/vault-snapshot/` role exists, when `ansible-playbook ansible/playbooks/day0-provision.yml` is run, then the snapshot role applies without errors.

2. **[Daily cron job is deployed]** Given the role is applied, when I inspect the vault service user's crontab (or `/etc/cron.d/vault-snapshot`) on the Vault VM, then a daily cron entry exists that runs both: `vault operator raft snapshot save /tmp/vault-snapshot.snap` and `rsync ... {snapshot_host}:/vault-snapshots/`.

3. **[Cron runs as vault service user]** Given the cron job is deployed, when I inspect the crontab, then the job runs as the `vault` service user (not root) — the vault service user has the minimum permissions required to take a snapshot and rsync.

4. **[snapshot_host is configurable via defaults]** Given `ansible/roles/vault-snapshot/defaults/main.yml`, when I read it, then `snapshot_host` variable is declared there (not hardcoded in tasks or templates) and `snapshot_host` is documented as the off-VM storage host (which MUST be a different physical host from the Vault VM per NFR10).

5. **[7-day pruning and 90-day monthly archive]** Given the cron job runs and snapshots accumulate, when I inspect the snapshot directory on `{snapshot_host}`, then:
   - Daily snapshots older than 7 days are pruned
   - Monthly archive snapshots (1st of each month) are retained for 90 days
   - Monthly archives are stored in a `monthly/` subdirectory or similar segregated path

6. **[snapshot_host is NOT the Vault VM (NFR10)]** Given the `snapshot_host` variable in defaults, when I inspect the configured value, then the IP or hostname resolves to a different physical host than `vault_vm`. This is enforced by documentation/convention, not runtime validation — a comment in `defaults/main.yml` must state this constraint.

## Tasks / Subtasks

- [ ] **Task 1: Create `ansible/roles/vault-snapshot/` structure** (AC: 1, 4)
  - [ ] Create:
    ```
    ansible/roles/vault-snapshot/
      defaults/main.yml
      tasks/main.yml
      templates/vault-snapshot.cron.j2
      templates/vault-snapshot.sh.j2
    ```

- [ ] **Task 2: Create `defaults/main.yml`** (AC: 4, 6)
  - [ ] Create `ansible/roles/vault-snapshot/defaults/main.yml`:
    ```yaml
    ---
    # vault-snapshot role defaults
    # D5 — Off-VM Raft snapshots
    # NFR10 — snapshot_host MUST be a different physical host from vault_vm

    snapshot_host: "PLACEHOLDER_SNAPSHOT_HOST"  # REQUIRED: set to off-VM storage host IP/hostname
    snapshot_dir: "/vault-snapshots"
    snapshot_tmp_path: "/tmp/vault-snapshot.snap"
    snapshot_daily_retention_days: 7
    snapshot_monthly_retention_days: 90
    vault_snapshot_service_user: vault
    vault_addr_local: "https://127.0.0.1:8200"
    vault_ca_cert_path: "/etc/vault.d/tls/ca.crt"
    ```

- [ ] **Task 3: Create `templates/vault-snapshot.sh.j2`** (AC: 2, 5)
  - [ ] Create the snapshot script template:
    ```bash
    #!/usr/bin/env bash
    # Vault Raft snapshot script — deployed by vault-snapshot Ansible role
    # D5: Off-VM snapshots; NFR10: snapshot_host must not be the Vault VM
    set -euo pipefail

    VAULT_ADDR="{{ vault_addr_local }}"
    VAULT_CACERT="{{ vault_ca_cert_path }}"
    SNAPSHOT_TMP="{{ snapshot_tmp_path }}"
    SNAPSHOT_HOST="{{ snapshot_host }}"
    SNAPSHOT_DIR="{{ snapshot_dir }}"
    DAILY_RETENTION={{ snapshot_daily_retention_days }}
    MONTHLY_RETENTION={{ snapshot_monthly_retention_days }}

    export VAULT_ADDR VAULT_CACERT

    # Take snapshot (requires Vault token in VAULT_TOKEN env — sourced from vault-agent or env file)
    vault operator raft snapshot save "${SNAPSHOT_TMP}"

    # Timestamp the snapshot
    TIMESTAMP=$(date +%Y%m%d-%H%M%S)
    DAILY_NAME="vault-snapshot-${TIMESTAMP}.snap"

    # Rsync to off-VM host — daily snapshots directory
    rsync -az "${SNAPSHOT_TMP}" "${SNAPSHOT_HOST}:${SNAPSHOT_DIR}/daily/${DAILY_NAME}"

    # Monthly archive: if today is the 1st, also copy to monthly/
    if [ "$(date +%d)" = "01" ]; then
      MONTHLY_NAME="vault-snapshot-monthly-${TIMESTAMP}.snap"
      rsync -az "${SNAPSHOT_TMP}" "${SNAPSHOT_HOST}:${SNAPSHOT_DIR}/monthly/${MONTHLY_NAME}"
    fi

    # Prune daily snapshots older than DAILY_RETENTION days
    ssh "${SNAPSHOT_HOST}" "find ${SNAPSHOT_DIR}/daily/ -name '*.snap' -mtime +${DAILY_RETENTION} -delete"

    # Prune monthly snapshots older than MONTHLY_RETENTION days
    ssh "${SNAPSHOT_HOST}" "find ${SNAPSHOT_DIR}/monthly/ -name '*.snap' -mtime +${MONTHLY_RETENTION} -delete"

    # Remove local temp file
    rm -f "${SNAPSHOT_TMP}"
    ```

- [ ] **Task 4: Create `tasks/main.yml`** (AC: 1, 2, 3)
  - [ ] Create `ansible/roles/vault-snapshot/tasks/main.yml`:
    ```yaml
    ---
    - name: Deploy vault snapshot script
      ansible.builtin.template:
        src: vault-snapshot.sh.j2
        dest: /usr/local/bin/vault-snapshot.sh
        owner: "{{ vault_snapshot_service_user }}"
        group: "{{ vault_snapshot_service_user }}"
        mode: "0750"

    - name: Ensure snapshot directories exist on snapshot_host
      ansible.builtin.command: >
        ssh {{ snapshot_host }}
        "mkdir -p {{ snapshot_dir }}/daily {{ snapshot_dir }}/monthly"
      changed_when: false

    - name: Deploy daily cron job for vault snapshot
      ansible.builtin.cron:
        name: vault-daily-snapshot
        user: "{{ vault_snapshot_service_user }}"
        hour: "2"
        minute: "0"
        job: >
          VAULT_TOKEN=$(cat /etc/vault.d/snapshot-token)
          /usr/local/bin/vault-snapshot.sh
          >> /var/log/vault/snapshot.log 2>&1
        state: present
    ```

- [ ] **Task 5: Add vault-snapshot role to day0-provision.yml** (AC: 1)
  - [ ] Update `ansible/playbooks/day0-provision.yml`:
    ```yaml
    roles:
      - vault-logrotate
      - vault-snapshot
    ```

- [ ] **Task 6: Provision snapshot token** (AC: 2)
  - [ ] The snapshot cron requires a Vault token with `snapshot` capability (`sys/storage/raft/snapshot`). Provision a dedicated snapshot policy and token:
    ```hcl
    # In vault-config or as a manual step:
    path "sys/storage/raft/snapshot" {
      capabilities = ["read"]
    }
    ```
  - [ ] Store the token at `/etc/vault.d/snapshot-token` on the VM (permissions: 0600, owned by vault user)
  - [ ] Document this in `docs/runbooks/` snapshot section

- [ ] **Task 7: Test snapshot cron (D5, NFR10)** (AC: 2, 5)
  - [ ] Manually invoke the script: `ssh vault_vm sudo -u vault /usr/local/bin/vault-snapshot.sh`
  - [ ] Confirm snapshot appears on `snapshot_host`: `ssh <snapshot_host> ls -la /vault-snapshots/daily/`
  - [ ] Confirm snapshot is off-VM (snapshot_host ≠ vault_vm IP) — NFR10

## Dev Notes

### Architecture Constraints

- **D5 — Off-VM Raft snapshots:** Architecture decision D5 mandates that snapshot destinations are NOT on the Vault VM. Snapshots stored only on the Vault VM would be lost in a blast-and-repave (Story 5.1). The `snapshot_host` var must be a different physical host.
  [Source: docs/terminus/infra/secrets/architecture.md#Operational Infrastructure — D5]

- **FR28 — Automated daily snapshot:** The snapshot must run automatically (cron/systemd timer), not only on-demand. The cron frequency is daily at a low-traffic time (2 AM local time — adjust per timezone).

- **NFR10 — Off-VM snapshot location:** "Raft snapshots stored off-VM (different physical host)." This is a hard constraint, not a preference. If `snapshot_host` is the same IP as the Vault VM, NFR10 is violated. Enforce by comment in defaults and verification in runbook.

- **Vault snapshot authentication:** `vault operator raft snapshot save` requires a token with `sys/storage/raft/snapshot` read capability. A dedicated snapshot service token stored at `/etc/vault.d/snapshot-token` (0600, vault-owned) is the cleanest approach. Alternatively, Vault Agent with AppRole auth could provide the token. Task 6 addresses this — don't skip it.

- **rsync SSH setup:** The vault service user on the Vault VM must have SSH key access to `snapshot_host` without password. Ensure the vault user's SSH public key is in `~/.ssh/authorized_keys` on `snapshot_host`. This is an operational prerequisite; document it in the automation runbook.

- **Monthly archive on 1st:** The `if [ "$(date +%d)" = "01" ]` check in the script ensures that on the 1st of each month, the same snapshot is also stored in `monthly/`. Monthly snapshots are retained 90 days (3 months of monthly archives).

### Project Structure Notes

- Creates `ansible/roles/vault-snapshot/` (new role directory)
- Updates `ansible/playbooks/day0-provision.yml` (adds vault-snapshot role)
- May require operator action: create snapshot policy in Vault and provision snapshot token

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 4.3]
- [Source: docs/terminus/infra/secrets/architecture.md#Operational Infrastructure — D5]
- [Source: docs/terminus/infra/secrets/architecture.md#Non-Functional Requirements — NFR10]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
