# Story 4.1: Ansible Control Structure and Inventory — Day-2 Operations Foundation

Status: ready-for-dev

## Story

As the operator,
I want a correctly structured Ansible control directory with inventory that targets the Vault VM by IP,
so that all Day-2 operations playbooks have a consistent, reliable execution foundation.

## Acceptance Criteria

1. **[Inventory file exists at canonical path]** Given the repository layout, when I inspect `ansible/inventory/hosts.yml`, then the file exists and contains a `vault_vm` host entry with an explicit IP address (sourced from `tofu output vm_ip`) or a variable reference to it.

2. **[Vault host variables are present]** Given `ansible/inventory/hosts.yml`, when I read the `vault_vm` host entry, then it includes at minimum:
   - `vault_addr` — the Vault API URL (e.g., `https://<vm_ip>:8200`)
   - `vault_ca_cert_path` — path on the VM to the Vault TLS CA certificate (e.g., `/etc/vault.d/tls/ca.crt`)

3. **[Ansible ping succeeds]** Given the inventory is correct and SSH access to the Vault VM is established, when I run `ansible -m ping vault_vm`, then the task returns `"ping": "pong"` with SUCCESS.

4. **[Role defaults are declared]** Given any Ansible role in `ansible/roles/`, when I inspect each role's `defaults/main.yml`, then it exists and contains all role-wide configurable variables with documented defaults — no hardcoded values inside `tasks/` or `playbooks/` that should be configurable.

5. **[Directory structure matches convention]** Given the `ansible/` directory, when I list its contents, then the structure contains at minimum:
   - `ansible/inventory/hosts.yml`
   - `ansible/roles/` (directory for role organization)
   - `ansible/playbooks/` (directory for top-level playbooks)
   - `ansible/ansible.cfg` (local configuration pointing to inventory)

6. **[ansible.cfg references local inventory]** Given `ansible/ansible.cfg`, when I read it, then it contains `inventory = inventory/hosts.yml` (or equivalent relative path) so that `ansible-playbook` commands run from `ansible/` use the correct inventory without `-i` flags.

## Tasks / Subtasks

- [ ] **Task 1: Create `ansible/` directory structure** (AC: 5)
  - [ ] Create:
    ```
    ansible/
      ansible.cfg
      inventory/
        hosts.yml
      roles/           (empty dir, populated by Stories 4.2 and 4.3)
      playbooks/       (contains verify-isolation.yml from Story 3.7)
    ```

- [ ] **Task 2: Create `ansible/ansible.cfg`** (AC: 6)
  - [ ] Create `ansible/ansible.cfg`:
    ```ini
    [defaults]
    inventory       = inventory/hosts.yml
    host_key_checking = True
    stdout_callback = yaml
    retry_files_enabled = False

    [ssh_connection]
    pipelining = True
    ```

- [ ] **Task 3: Create `ansible/inventory/hosts.yml`** (AC: 1, 2)
  - [ ] Create `ansible/inventory/hosts.yml`:
    ```yaml
    ---
    all:
      hosts:
        vault_vm:
          ansible_host: "PLACEHOLDER_IP"  # Set from: cd ../terraform && tofu output -raw vm_ip
          ansible_user: vault             # SSH user — review per Proxmox VM provisioning (Story 1.2)
          vault_addr: "https://{{ ansible_host }}:8200"
          vault_ca_cert_path: "/etc/vault.d/tls/ca.crt"
    ```
  - [ ] **Critical:** Replace `PLACEHOLDER_IP` with the actual VM IP from `tofu output -raw vm_ip`. Gate this on Story 1.2 being complete. If the VM is not yet provisioned, the hosts.yml can use the placeholder with a comment.
  - [ ] If the VM user is `ubuntu` or `root` instead of `vault`, update `ansible_user` accordingly.

- [ ] **Task 4: Verify Ansible can reach the vault_vm** (AC: 3)
  - [ ] With actual IP populated and SSH key accessible: `cd ansible && ansible -m ping vault_vm`
  - [ ] Confirm: `vault_vm | SUCCESS => { "ping": "pong" }`

- [ ] **Task 5: Ensure roles have defaults/main.yml** (AC: 4)
  - [ ] For any other roles created in this or concurrent stories (vault-logrotate from 4.2, vault-snapshot from 4.3):
    - Each role MUST have `defaults/main.yml` with all configurable variables declared
    - Variables should NOT be hardcoded inside `tasks/main.yml` directly
  - [ ] Create placeholder role directories if needed:
    ```
    ansible/roles/vault-logrotate/
      defaults/main.yml
      tasks/main.yml
    ansible/roles/vault-snapshot/
      defaults/main.yml
      tasks/main.yml
    ```

- [ ] **Task 6: Verify verify-isolation.yml from Story 3.7 works with this inventory** (AC: 3)
  - [ ] If Story 3.7 created a stub inventory, reconcile it with this story's hosts.yml
  - [ ] Run: `ansible-playbook ansible/playbooks/verify-isolation.yml` → confirm all checks pass using the canonical inventory

## Dev Notes

### Architecture Constraints

- **Ansible version:** The architecture baseline uses `ansible-core 2.19`. Ensure the installed version is at least 2.19 on the operator machine: `ansible --version`.

- **SSH access to Vault VM:** The Vault VM was provisioned in Story 1.2 (Proxmox VM). The SSH key used for VM provisioning (via the Proxmox module) must be the same key available to the Ansible control host. Confirm the VM's `ansible_user` matches how the SSH key was injected during provisioning (typically the OS image's default user or a dedicated `vault` user).

- **Host variables vs group_vars:** For a single-host inventory this size, host variables directly in `hosts.yml` are appropriate. If/when multiple hosts are onboarded, group_vars should be introduced. Do not over-engineer for future growth now.

- **`vault_addr` variable in inventory:** The `vault_addr` host variable is used by Story 3.7's `verify-isolation.yml` and all future Day-2 playbooks. Establishing this pattern here ensures all playbooks can reference `{{ vault_addr }}` without repeating the URL construction.

- **vault_ca_cert_path:** This is the path ON THE VM (not the operator machine) to the TLS CA cert generated in Story 1.3. The cert is at `/etc/vault.d/tls/ca.crt` (or equivalent path established in Stories 1.3 and 1.5). Confirm the actual path by checking the vault.hcl configuration.

### Project Structure Notes

- Creates `ansible/` directory tree (new directories)
- Creates `ansible/ansible.cfg` (new file)
- Creates `ansible/inventory/hosts.yml` (new file)
- May create `ansible/roles/vault-logrotate/` and `ansible/roles/vault-snapshot/` stubs (for Stories 4.2 and 4.3)

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 4.1]
- [Source: docs/terminus/infra/secrets/architecture.md#Operational Infrastructure — Ansible]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
