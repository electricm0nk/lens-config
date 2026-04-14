# Story 4.2: Vault Audit Log Rotation — logrotate Role via Ansible (D7, FR35, NFR14)

Status: ready-for-dev

## Story

As the operator,
I want the Vault audit log at `/var/log/vault/audit.log` automatically rotated via a logrotate configuration deployed by Ansible,
so that the audit log disk footprint is bounded and historical logs are compressed and retained for 90 days.

## Acceptance Criteria

1. **[logrotate role exists and is applied]** Given the `ansible/roles/vault-logrotate/` role exists, when I run `ansible-playbook ansible/playbooks/day0-provision.yml`, then the role applies the logrotate configuration to the Vault VM without errors.

2. **[logrotate config deployed at canonical path]** Given the role is applied, when I inspect `/etc/logrotate.d/vault-audit` on the Vault VM, then the file exists with the following minimum attributes:
   - `daily` rotation
   - `rotate 90` (retain 90 days of rotated files)
   - `compress`
   - `delaycompress` (do not compress the most recently rotated file — Vault may still have it open briefly)
   - `maxsize 100M` (trigger rotation if the file exceeds 100MB regardless of schedule)
   - `postrotate` containing `kill -HUP $(cat /var/run/vault.pid 2>/dev/null || systemctl show vault --property=MainPID | cut -d= -f2 2>/dev/null)` or equivalent SIGHUP to Vault

3. **[logrotate dry run succeeds]** Given the logrotate config is deployed, when I run `logrotate -d /etc/logrotate.d/vault-audit` on the Vault VM, then the command completes without errors (dry run validates the configuration syntax and logic).

4. **[Rotation produces compressed log file]** Given the logrotate config is deployed and `/var/log/vault/audit.log` contains data, when I force-rotate with `logrotate -f /etc/logrotate.d/vault-audit`, then a rotated, compressed log file exists (e.g., `/var/log/vault/audit.log.1` or `.gz` depending on `delaycompress` settings) and a fresh empty `audit.log` is present for Vault to continue writing.

5. **[Vault continues writing after SIGHUP]** Given log rotation has been triggered (forcerotate or scheduled), when I perform a Vault operation (e.g., `vault status`), then within 30 seconds a new entry appears in the fresh `audit.log` — confirming Vault has reopened the file after SIGHUP.

6. **[Role defaults contain all configurable parameters]** Given `ansible/roles/vault-logrotate/defaults/main.yml`, when I read it, then it contains defaults for: `vault_audit_log_path`, `vault_log_rotate_count`, `vault_log_rotate_maxsize`, none of which are hardcoded in `tasks/main.yml`.

## Tasks / Subtasks

- [ ] **Task 1: Create `ansible/roles/vault-logrotate/` structure** (AC: 1, 6)
  - [ ] Create:
    ```
    ansible/roles/vault-logrotate/
      defaults/main.yml
      tasks/main.yml
      templates/vault-audit.logrotate.j2
    ```

- [ ] **Task 2: Create `defaults/main.yml`** (AC: 6)
  - [ ] Create `ansible/roles/vault-logrotate/defaults/main.yml`:
    ```yaml
    ---
    # vault-logrotate role defaults
    # D7 — Audit log retention: 90 days (NFR14 operational documentation)
    vault_audit_log_path: /var/log/vault/audit.log
    vault_log_rotate_count: 90
    vault_log_rotate_maxsize: 100M
    vault_logrotate_conf_path: /etc/logrotate.d/vault-audit
    ```

- [ ] **Task 3: Create `templates/vault-audit.logrotate.j2`** (AC: 2)
  - [ ] Create the Jinja2 template:
    ```
    {{ vault_audit_log_path }} {
        daily
        rotate {{ vault_log_rotate_count }}
        compress
        delaycompress
        missingok
        notifempty
        maxsize {{ vault_log_rotate_maxsize }}
        postrotate
            # Send SIGHUP to Vault to reopen audit log file handle
            systemctl kill --signal=HUP vault.service 2>/dev/null || true
        endscript
    }
    ```
  - [ ] Note: `systemctl kill --signal=HUP vault.service` is the preferred form for systemd-managed services vs PID file lookup

- [ ] **Task 4: Create `tasks/main.yml`** (AC: 1, 2)
  - [ ] Create `ansible/roles/vault-logrotate/tasks/main.yml`:
    ```yaml
    ---
    - name: Deploy vault audit log rotation configuration
      ansible.builtin.template:
        src: vault-audit.logrotate.j2
        dest: "{{ vault_logrotate_conf_path }}"
        owner: root
        group: root
        mode: "0644"
      notify: Reload logrotate

    - name: Ensure logrotate is installed
      ansible.builtin.package:
        name: logrotate
        state: present
    ```

- [ ] **Task 5: Create or update `ansible/playbooks/day0-provision.yml`** (AC: 1)
  - [ ] Create `ansible/playbooks/day0-provision.yml` (if not already existing):
    ```yaml
    ---
    - name: Day-0 VM Provisioning
      hosts: vault_vm
      become: true
      roles:
        - vault-logrotate
    ```
  - [ ] If the file already exists (from Story 1.5 or 1.8), add the `vault-logrotate` role to the role list

- [ ] **Task 6: Test logrotate deployment**  
  - [ ] Run: `ansible-playbook ansible/playbooks/day0-provision.yml --tags logrotate` (or run full playbook)
  - [ ] Verify file exists on VM: `ssh vault_vm cat /etc/logrotate.d/vault-audit` → shows correct config (AC: 2)
  - [ ] Run dry run: `ssh vault_vm sudo logrotate -d /etc/logrotate.d/vault-audit` → no errors (AC: 3)
  - [ ] Force rotation: `ssh vault_vm sudo logrotate -f /etc/logrotate.d/vault-audit` → inspect `/var/log/vault/` for rotated file (AC: 4)
  - [ ] Confirm Vault still writing: `ssh vault_vm tail -1 /var/log/vault/audit.log` shows new entry within 30s of Vault operation (AC: 5)

## Dev Notes

### Architecture Constraints

- **D7 — Audit log rotation:** Architecture decision D7 mandates: "logrotate configured on the Vault VM: daily rotation, 90-day retention, maxsize 100M, postrotate SIGHUP to Vault." These exact values must match `defaults/main.yml`. Do not change `rotate 90` to a smaller value.
  [Source: docs/terminus/infra/secrets/architecture.md#Operational Infrastructure — D7]

- **FR35 — Audit log must remain SIEM-compatible:** The audit log format is structured JSON (per Story 3.1). Compression preserves the content; `delaycompress` ensures Vault is not writing to a file while it is being gzip-compressed. The `missingok` directive prevents logrotate errors if the audit log doesn't exist yet (during bootstrapping).

- **NFR14 — Documented operational procedures:** The Day-0 provision playbook referenced in this story is itself an operational procedure artifact. Ensure the playbook has a comment header describing its purpose and expected runtime.

- **SIGHUP semantics:** When Vault receives SIGHUP, it re-reads the vault.hcl configuration (only certain fields, not Raft storage) AND reopens log file descriptors. This is the correct signal for triggering audit log file handle rotation. Using `systemctl kill --signal=HUP vault.service` is safer than PID-file-based approaches because it works even if `/var/run/vault.pid` doesn't exist.

- **`logrotate -d` for testing:** Always run `logrotate -d` (debug/dry-run) before `logrotate -f` (force) in production. The `-d` flag validates the config and shows what WOULD happen without making changes.

### Project Structure Notes

- Creates `ansible/roles/vault-logrotate/` (new role directory with defaults, tasks, templates)
- Creates or updates `ansible/playbooks/day0-provision.yml` (may exist from earlier epic; add role if so)

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 4.2]
- [Source: docs/terminus/infra/secrets/architecture.md#Operational Infrastructure — D7]
- [Source: docs/terminus/infra/secrets/architecture.md#Additional Requirements — FR35]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
