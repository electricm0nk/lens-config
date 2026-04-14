# Story 4.4: Manual Secret Rotation Procedure — Ansible-Assisted Rotation Workflow (FR23)

Status: ready-for-dev

## Story

As the operator,
I want an Ansible playbook `day2-rotate-secrets.yml` that automates the manual secret rotation workflow including generating new credentials, writing them to Vault KV v2, updating the corresponding SOPS file, and committing the change,
so that secret rotation is repeatable, auditable, and consistent with the SOPS/Vault sync established in Epic 2.

## Acceptance Criteria

1. **[Rotation playbook exists and accepts target path]** Given `ansible/playbooks/day2-rotate-secrets.yml` exists, when I run `ansible-playbook ansible/playbooks/day2-rotate-secrets.yml -e "target_path=terminus/infra/postgres/app_role_password"`, then the playbook runs without syntax errors and enters the rotation workflow.

2. **[New credential generated and written to Vault (FR23)]** Given the rotation playbook runs for `target_path=terminus/infra/postgres/app_role_password`, when the playbook completes successfully, then a new value is written to that path in Vault KV v2 (a new version is created) and `vault kv get secret/terminus/infra/postgres/app_role_password` returns the newly rotated value.

3. **[Old credential retained in version history]** Given the rotation playbook runs, when I list version history with `vault kv metadata get secret/terminus/infra/postgres/app_role_password`, then the versions list shows at least 2 versions — the new version AND the previous version (which is not destroyed — retained for audit trail).

4. **[SOPS file updated and committed]** Given a rotation for `postgres/app_role_password`, when the playbook completes, when I run `git log --oneline -1`, then a new commit exists with message following the `chore(rotation): rotate <target_path> credentials` convention, and `sops/postgres.sops.yaml` contains the new value (decryptable with the age key).

5. **[New credential verified readable]** Given the rotation is complete and the SOPS file is committed, when I run `sops -d sops/postgres.sops.yaml | grep <key>`, then the key shows the new value (same value as what was written to Vault).

6. **[TTL constraint respected (NFR3)]** Given the rotation playbook, when I read its `## TTL Constraints` documentation, then it notes that rotated credentials should have a TTL ≤ 90 days (NFR3) applied at the consuming service — the rotation procedure itself does not enforce TTL on Vault KV v2 static secrets, but documents the expectation.

7. **[CLI fallback procedure documented]** Given `docs/runbooks/secret-rotation.md` exists, when I read it, then it contains both an "Ansible-Assisted Rotation" section (the playbook procedure) and a "Manual CLI Fallback" section describing how to rotate using `vault kv put` + `sops -e` + `git commit` without Ansible.

## Tasks / Subtasks

- [ ] **Task 1: Create `ansible/playbooks/day2-rotate-secrets.yml`** (AC: 1, 2, 3, 4)
  - [ ] Create `ansible/playbooks/day2-rotate-secrets.yml`:
    ```yaml
    ---
    # Day-2 Secret Rotation Playbook
    # Usage: ansible-playbook day2-rotate-secrets.yml -e "target_path=terminus/infra/postgres/app_role_password"
    # FR23: Manual rotation procedure; NFR3: new credentials must have TTL <= 90 days at the consumer

    - name: Vault Secret Rotation — Ansible-Assisted
      hosts: localhost
      gather_facts: false

      vars:
        # Override with -e "target_path=<path>" — format: <namespace>/<service>/<key>
        target_path: ""

      pre_tasks:
        - name: Fail if target_path not provided
          ansible.builtin.fail:
            msg: "target_path must be provided: -e 'target_path=terminus/infra/postgres/app_role_password'"
          when: target_path == ""

      tasks:
        - name: "STEP 1 — Read current secret from Vault (establish baseline)"
          ansible.builtin.command: >
            vault kv get -format=json secret/{{ target_path }}
          register: current_secret_raw
          environment:
            VAULT_ADDR: "{{ lookup('env', 'VAULT_ADDR') }}"
            VAULT_TOKEN: "{{ lookup('env', 'VAULT_TOKEN') }}"
          changed_when: false

        - name: "STEP 1 — Parse current version number"
          ansible.builtin.set_fact:
            current_version: "{{ (current_secret_raw.stdout | from_json).data.metadata.version }}"

        - name: "STEP 2 — Generate new credential (simple random string)"
          ansible.builtin.command: >
            openssl rand -hex 32
          register: new_credential
          changed_when: false

        - name: "STEP 3 — Write new credential to Vault KV v2"
          ansible.builtin.command: >
            vault kv put secret/{{ target_path }} value="{{ new_credential.stdout }}"
          environment:
            VAULT_ADDR: "{{ lookup('env', 'VAULT_ADDR') }}"
            VAULT_TOKEN: "{{ lookup('env', 'VAULT_TOKEN') }}"
          register: vault_write_result

        - name: "STEP 3 — Verify new version created"
          ansible.builtin.command: >
            vault kv get -format=json secret/{{ target_path }}
          register: new_secret_raw
          environment:
            VAULT_ADDR: "{{ lookup('env', 'VAULT_ADDR') }}"
            VAULT_TOKEN: "{{ lookup('env', 'VAULT_TOKEN') }}"
          changed_when: false

        - name: "STEP 3 — Assert version incremented"
          ansible.builtin.assert:
            that:
              - "(new_secret_raw.stdout | from_json).data.metadata.version | int > current_version | int"
            fail_msg: "Version did not increment after write — rotation may have failed"
            success_msg: "Version incremented from {{ current_version }} to {{ (new_secret_raw.stdout | from_json).data.metadata.version }}"

        - name: "STEP 4 — Determine SOPS file path from target_path"
          ansible.builtin.set_fact:
            # target_path: terminus/infra/postgres/app_role_password
            # SOPS path: sops/postgres.sops.yaml (service-level file)
            sops_service: "{{ target_path.split('/')[2] }}"
            sops_file: "sops/{{ target_path.split('/')[2] }}.sops.yaml"

        - name: "STEP 4 — Update SOPS file with new value"
          ansible.builtin.command: >
            sops --set '["{{ target_path.split("/")[-1] }}"] "{{ new_credential.stdout }}"' {{ sops_file }}
          environment:
            SOPS_AGE_KEY_FILE: "{{ lookup('env', 'SOPS_AGE_KEY_FILE') }}"

        - name: "STEP 5 — Commit SOPS file update"
          ansible.builtin.shell: |
            git add {{ sops_file }}
            git commit -m "chore(rotation): rotate {{ target_path }} credentials"

        - name: "STEP 6 — Verify new credential readable from SOPS (consistency check)"
          ansible.builtin.command: >
            sops -d {{ sops_file }}
          environment:
            SOPS_AGE_KEY_FILE: "{{ lookup('env', 'SOPS_AGE_KEY_FILE') }}"
          register: sops_verify
          changed_when: false

        - name: "ROTATION COMPLETE"
          ansible.builtin.debug:
            msg: >
              Secret rotation complete for {{ target_path }}.
              New Vault version: {{ (new_secret_raw.stdout | from_json).data.metadata.version }}.
              SOPS file {{ sops_file }} updated and committed.
    ```

- [ ] **Task 2: Test rotation end-to-end** (AC: 2, 3, 4, 5)
  - [ ] Ensure `VAULT_ADDR`, `VAULT_TOKEN`, and `SOPS_AGE_KEY_FILE` are exported in the shell
  - [ ] Run: `ansible-playbook ansible/playbooks/day2-rotate-secrets.yml -e "target_path=terminus/infra/postgres/app_role_password"`
  - [ ] Verify new version in Vault: `vault kv get secret/terminus/infra/postgres/app_role_password` → shows new value (AC: 2)
  - [ ] Verify old version retained: `vault kv metadata get secret/terminus/infra/postgres/app_role_password` → shows 2+ versions (AC: 3)
  - [ ] Verify git commit: `git log --oneline -1` → shows `chore(rotation): rotate terminus/infra/postgres/app_role_password credentials` (AC: 4)
  - [ ] Verify SOPS: `sops -d sops/postgres.sops.yaml | grep app_role_password` → shows new value (AC: 5)

- [ ] **Task 3: Create `docs/runbooks/secret-rotation.md`** (AC: 6, 7)
  - [ ] Create the runbook with sections:
    - `# Secret Rotation Runbook`
    - `## Overview` — when to rotate, rotation frequency, NFR3 TTL reminder
    - `## Ansible-Assisted Rotation` — the `day2-rotate-secrets.yml` invocation with examples
    - `## Manual CLI Fallback` — step-by-step: `vault kv put` → `sops -e` update → git commit → verify
    - `## TTL Constraints` — "Vault KV v2 static secrets have no built-in TTL; enforce rotation schedule via ops-log.md entries. Target rotation frequency: every 90 days minimum (NFR3)."
    - `## Version History` — how to read version history and retrieve rotated credentials
    - `## Growth: Automated Rotation` — see Story 4.5 (placeholder for automated hook design)

## Dev Notes

### Architecture Constraints

- **FR23 — Manual rotation procedure:** The rotation playbook is the primary mechanism for manual secret rotation. It must produce: new Vault KV v2 version, SOPS file update, git commit. All three are required for a complete rotation.

- **Old version retention:** The old credential version is NOT destroyed during rotation. It is retained with `destroyed: false` in Vault KV v2 metadata. This enables rollback if the new credential is incorrect and provides an audit trail. Operators can manually destroy old versions after the new credential is confirmed working (per Story 2.5 procedures) — but this is NOT done automatically during rotation.

- **SOPS `--set` syntax:** The `sops --set` command updates a specific key within an existing SOPS-encrypted file. The syntax `sops --set '["key"] "value"' file.sops.yaml` modifies only the specified key without decrypting and re-encrypting the entire file. This is the correct idiom for targeted updates.

- **Credential generation:** The playbook uses `openssl rand -hex 32` to generate a 64-character hex credential. For production use, the generation method should match the credential type (database password vs API key vs random token). The playbook's generation step is intentionally simplistic — operators may override with service-specific generation logic as needed.

- **Environment variable prerequisites:** The playbook runs on `localhost` and requires `VAULT_ADDR`, `VAULT_TOKEN`, and `SOPS_AGE_KEY_FILE` in the shell environment. Document these prerequisites explicitly in the runbook.

### Project Structure Notes

- Creates `ansible/playbooks/day2-rotate-secrets.yml` (new file)
- Creates `docs/runbooks/secret-rotation.md` (new file)

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 4.4]
- [Source: docs/terminus/infra/secrets/architecture.md#Additional Requirements — FR23]
- [Source: docs/terminus/infra/secrets/architecture.md#Non-Functional Requirements — NFR3]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
