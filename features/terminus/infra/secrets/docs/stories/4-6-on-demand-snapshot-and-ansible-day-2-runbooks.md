# Story 4.6: On-Demand Snapshot and Day-2 Runbooks Completeness Check (FR27, NFR14, NFR17)

Status: ready-for-dev

## Story

As the operator,
I want an on-demand Vault Raft snapshot playbook and a complete set of 8 operational runbooks,
so that any operator can perform any Day-2 or recovery operation using documented, tested procedures.

## Acceptance Criteria

1. **[On-demand snapshot playbook exists and runs]** Given `ansible/playbooks/day2-snapshot.yml` exists, when I run `ansible-playbook ansible/playbooks/day2-snapshot.yml`, then the playbook runs `vault operator raft snapshot save`, rsync's the snapshot to `snapshot_host`, and reports the snapshot file path and size on `snapshot_host`.

2. **[On-demand snapshot reaches off-VM host (D5)]** Given the playbook succeeds, when I ssh to `snapshot_host` and inspect the snapshot directory, then a snapshot file with today's timestamp exists and is non-zero size.

3. **[All 8 runbooks exist at canonical paths]** Given the `docs/runbooks/` directory, when I list its contents, then ALL of the following 8 files exist:
   - `docs/runbooks/day0-bootstrap.md` (from Story 1.8)
   - `docs/runbooks/break-glass.md` (from Story 5.3)
   - `docs/runbooks/blast-and-repave.md` (from Story 5.1)
   - `docs/runbooks/snapshot-restore.md` (from Story 5.2)
   - `docs/runbooks/secret-authoring.md` (from Story 2.3)
   - `docs/runbooks/workload-onboarding.md` (from Story 3.8)
   - `docs/runbooks/age-key-custody.md` (from Story 2.1)
   - `docs/runbooks/secret-rotation.md` (from Story 4.4)

4. **[Each runbook has required sections]** Given each of the 8 runbooks, when I inspect them, then every runbook contains at minimum: a `# Title` header, an `## Overview` section, and at least one numbered procedure section.

5. **[NFR17 — All runbooks directly executable]** Given any runbook from the 8 listed, when an operator follows the documented steps, then no step requires undocumented knowledge unavailable in the runbook or its references — all commands are complete and all prerequisite environment variables are documented.

6. **[Cross-references are valid]** Given any runbook that references another document (e.g., break-glass.md references ops-log.md), when I inspect those cross-referenced documents, then the referenced files exist at the stated paths.

## Tasks / Subtasks

- [ ] **Task 1: Create `ansible/playbooks/day2-snapshot.yml`** (AC: 1, 2)
  - [ ] Create `ansible/playbooks/day2-snapshot.yml`:
    ```yaml
    ---
    # Day-2 On-Demand Snapshot Playbook
    # FR27: On-demand Raft snapshot capability; D5: snapshot must reach off-VM host
    # Usage: ansible-playbook day2-snapshot.yml

    - name: Vault On-Demand Raft Snapshot
      hosts: vault_vm
      gather_facts: false
      become: true
      become_user: vault

      vars:
        snapshot_timestamp: "{{ lookup('pipe', 'date +%Y%m%d-%H%M%S') }}"
        snapshot_tmp_path: "/tmp/vault-snapshot-{{ snapshot_timestamp }}.snap"
        snapshot_host: "{{ hostvars['vault_vm']['snapshot_host'] | default(lookup('env', 'SNAPSHOT_HOST')) }}"
        snapshot_dir: "/vault-snapshots/on-demand"

      tasks:
        - name: "STEP 1 — Take Vault Raft snapshot"
          ansible.builtin.command: >
            vault operator raft snapshot save {{ snapshot_tmp_path }}
          environment:
            VAULT_ADDR: "{{ vault_addr }}"
            VAULT_CACERT: "{{ vault_ca_cert_path }}"
            VAULT_TOKEN: "{{ lookup('env', 'VAULT_TOKEN') }}"
          register: snapshot_result

        - name: "STEP 2 — Verify snapshot file exists and is non-zero"
          ansible.builtin.stat:
            path: "{{ snapshot_tmp_path }}"
          register: snapshot_stat

        - name: "STEP 2 — Assert snapshot is valid"
          ansible.builtin.assert:
            that:
              - snapshot_stat.stat.exists
              - snapshot_stat.stat.size > 0
            fail_msg: "Snapshot file is empty or does not exist — aborting rsync"
            success_msg: "Snapshot file size: {{ snapshot_stat.stat.size }} bytes"

        - name: "STEP 3 — Create on-demand snapshot directory on snapshot_host"
          ansible.builtin.command: >
            ssh {{ snapshot_host }} "mkdir -p {{ snapshot_dir }}"
          changed_when: false

        - name: "STEP 4 — Rsync snapshot to off-VM host (D5, NFR10)"
          ansible.builtin.command: >
            rsync -az {{ snapshot_tmp_path }} {{ snapshot_host }}:{{ snapshot_dir }}/
          register: rsync_result

        - name: "STEP 5 — Remove local temp snapshot"
          ansible.builtin.file:
            path: "{{ snapshot_tmp_path }}"
            state: absent

        - name: "STEP 6 — Report snapshot location"
          ansible.builtin.debug:
            msg: >
              On-demand snapshot complete.
              File: {{ snapshot_dir }}/{{ snapshot_tmp_path | basename }}
              Host: {{ snapshot_host }}
              Size: {{ snapshot_stat.stat.size }} bytes
    ```

- [ ] **Task 2: Audit runbook completeness — verify all 8 exist or are stubs** (AC: 3, 4)
  - [ ] Check each of the 8 runbooks:
    - `docs/runbooks/day0-bootstrap.md` — created in Story 1.8? If not, create a stub
    - `docs/runbooks/secret-authoring.md` — created in Story 2.3? If not, create a stub
    - `docs/runbooks/age-key-custody.md` — created in Story 2.1? If not, create a stub
    - `docs/runbooks/workload-onboarding.md` — created in Story 3.8? If not, create a stub
    - `docs/runbooks/secret-rotation.md` — created in Story 4.4? If not, create a stub
    - `docs/runbooks/tls-renewal.md` — created in Story 4.7? If not, create a stub
    - `docs/runbooks/break-glass.md` — to be created in Story 5.3; create a stub: `# Break-Glass Procedure\n\n## Status\n\nSTUB — See Story 5.3 for full implementation.`
    - `docs/runbooks/blast-and-repave.md` — to be created in Story 5.1; create a stub
    - `docs/runbooks/snapshot-restore.md` — to be created in Story 5.2; create a stub
  - [ ] For each stub, ensure it has at minimum: title header, overview, and "See Story X.X for full content" note

- [ ] **Task 3: Validate cross-references** (AC: 6)
  - [ ] For each of the 8 runbooks, grep for any cross-references to other files:
    `grep -r "docs/" docs/runbooks/` 
  - [ ] Confirm each referenced file path exists
  - [ ] Confirm `docs/ops-log.md` exists (referenced by break-glass.md in Story 5.3) — create a stub if not:
    ```markdown
    # Operations Log

    This file records all break-glass access sessions, compliance deviations, and significant
    operational events. Each entry must be created during or immediately after the operation.

    ## Entry Format

    ## <Operation Type> — <ISO Timestamp>
    - **Operator:** <name>
    - **Reason:** <brief justification>
    - **Operations Performed:** <list>
    - **Revocation/Cleanup:** <confirmation>
    - **Vault Audit Reference:** <audit log timestamp range>
    ```

- [ ] **Task 4: Verify snapshot playbook runs successfully** (AC: 1, 2)
  - [ ] Ensure `VAULT_TOKEN` and `SNAPSHOT_HOST` are set in environment (or use inventory vars)
  - [ ] Run: `ansible-playbook ansible/playbooks/day2-snapshot.yml`
  - [ ] Confirm snapshot appears on `snapshot_host`: `ssh <snapshot_host> ls -la /vault-snapshots/on-demand/`
  - [ ] Confirm file is non-zero size (AC: 2)

- [ ] **Task 5: Run runbook NFR17 review** (AC: 5)
  - [ ] For 3 of the 8 runbooks (representative check), walk through the procedure as a new operator
  - [ ] Confirm: all prerequisite environment variables are listed at the top of the runbook
  - [ ] Confirm: all commands are complete and copyable (no placeholders without explanation)
  - [ ] Note any gaps and fill them

## Dev Notes

### Architecture Constraints

- **FR27 — On-demand snapshot capability:** An on-demand snapshot is needed for situations like: before a `tofu apply` that changes Vault configuration, before break-glass access, before any high-risk operational change. The `day2-snapshot.yml` playbook provides this capability independently of the daily automated cron (Story 4.3).

- **NFR14 — All automated Day-2 operations have runbooks:** This story explicitly verifies that all 8 required runbooks exist. The verification step (Task 2) catches any runbooks that were marked as "will create" in earlier stories but were not yet completed. Create stubs now; full content is provided by their respective stories.

- **NFR17 — Runbooks must be directly executable:** The quality bar for runbooks is "any qualified operator can execute without tribal knowledge." Runbooks that rely on unwritten context fail this criterion. Task 5's walkthrough is a lightweight compliance check.

- **ops-log.md is a first-class operational document:** Referenced by break-glass.md (Story 5.3), compliance-deviation records (Story 5.4), and TLS renewal (Story 4.7). Creating the stub here ensures the cross-references are valid before those stories are implemented.

- **Story ordering note:** This story (4.6) is the last Epic 4 story in execution order (per the sprint-status.yaml ordering which puts 4.7 before 4.6). It serves as an integration/completeness check for all Epic 4 artifacts. The on-demand snapshot is new; the runbook audit is verification of prior work.

### Project Structure Notes

- Creates `ansible/playbooks/day2-snapshot.yml` (new file)
- Creates `docs/ops-log.md` stub (new file, permanent artifact)
- Creates any missing runbook stubs in `docs/runbooks/` (temporary stubs to be replaced by their owning stories)

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 4.6]
- [Source: docs/terminus/infra/secrets/architecture.md#Additional Requirements — FR27/NFR14/NFR17]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
