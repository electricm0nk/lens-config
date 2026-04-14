# Story 3.7: Isolation Verification Suite — ansible-playbook verify-isolation.yml (AR-S4 HIGH, D8)

**Sprint Planning Note:** This story must be developed CONCURRENTLY with Stories 3.3, 3.4, and 3.5 (not sequentially after them). As each AppRole identity is provisioned, a corresponding isolation check should be added to the playbook. All three policy stories and this verification story share a sprint — build the checks incrementally.

Status: ready-for-dev

## Story

As the operator,
I want an idempotent Ansible playbook `verify-isolation.yml` that validates all three AppRole policy isolation constraints,
so that isolation can be verified on demand and integrated into post-apply CI checks without manual intervention.

## Acceptance Criteria

1. **[Playbook runs to completion]** Given all three AppRole identities are provisioned (Stories 3.3, 3.4, 3.5), when I run `ansible-playbook ansible/playbooks/verify-isolation.yml`, then the playbook completes with zero Ansible task failures.

2. **[Three checks execute and all PASS]** Given the playbook is run against a correctly isolated Vault, when I observe the playbook output, then I see three distinct check results:
   - Check 1: `opentofu READ secret/terminus/infra/postgres/app_role_password — PASS`
   - Check 2: `openclaw READ secret/terminus/agent/openclaw/api_key — PASS (own namespace)`
   - Check 3 variant: `openclaw READ secret/terminus/infra/postgres/app_role_password — PASS (403 expected, access denied as required)`

3. **[Summary line indicates all passed]** Given all three checks pass, when the playbook completes, then the stdout contains the exact line: `ISOLATION VERIFICATION: ALL CHECKS PASSED`

4. **[Non-zero exit on failure]** Given a policy misconfiguration (e.g., openclaw can accidentally read infra paths), when the playbook runs, then it exits with a non-zero exit code (fail_when ensures CI pipelines detect the failure).

5. **[Playbook is idempotent]** Given the playbook was run once successfully, when I run it a second time without any infrastructure changes, then it produces the same results with no state changes in Vault — the checks are read-only and do not modify Vault state.

6. **[Audit log records all check operations]** Given the playbook performs vault operations (reads and denied reads), when I inspect the Vault audit log after the playbook run, then entries for all playbook operations appear (confirms FR17 — all ops logged, including automated verification runs).

## Tasks / Subtasks

- [ ] **Task 1: Create `ansible/playbooks/verify-isolation.yml`** (AC: 1, 2, 3, 4, 5)
  - [ ] Create `ansible/playbooks/verify-isolation.yml`:
    ```yaml
    ---
    - name: Vault AppRole Isolation Verification
      hosts: vault_vm
      gather_facts: false
      vars_prompt:
        - name: opentofu_role_id
          prompt: "OpenTofu AppRole role_id"
          private: false
        - name: opentofu_secret_id
          prompt: "OpenTofu AppRole secret_id"
          private: true
        - name: openclaw_role_id
          prompt: "OpenClaw AppRole role_id"
          private: false
        - name: openclaw_secret_id
          prompt: "OpenClaw AppRole secret_id"
          private: true

      tasks:
        - name: "CHECK 1 — OpenTofu reads infra/postgres path (expected: 200 OK)"
          ansible.builtin.uri:
            url: "{{ vault_addr }}/v1/auth/approle/login"
            method: POST
            body_format: json
            body:
              role_id: "{{ opentofu_role_id }}"
              secret_id: "{{ opentofu_secret_id }}"
            ca_path: "{{ vault_ca_cert_path }}"
            status_code: 200
          register: opentofu_login

        - name: "CHECK 1 — OpenTofu reads infra/postgres/app_role_password"
          ansible.builtin.uri:
            url: "{{ vault_addr }}/v1/secret/data/terminus/infra/postgres/app_role_password"
            method: GET
            headers:
              X-Vault-Token: "{{ opentofu_login.json.auth.client_token }}"
            ca_path: "{{ vault_ca_cert_path }}"
            status_code: 200
          register: check1_result

        - name: "CHECK 1 — PASS"
          ansible.builtin.debug:
            msg: "opentofu READ secret/terminus/infra/postgres/app_role_password — PASS"
          when: check1_result.status == 200

        - name: "CHECK 2 — OpenClaw reads own namespace agent/openclaw/api_key (expected: 200 OK)"
          ansible.builtin.uri:
            url: "{{ vault_addr }}/v1/auth/approle/login"
            method: POST
            body_format: json
            body:
              role_id: "{{ openclaw_role_id }}"
              secret_id: "{{ openclaw_secret_id }}"
            ca_path: "{{ vault_ca_cert_path }}"
            status_code: 200
          register: openclaw_login

        - name: "CHECK 2 — OpenClaw reads own secret"
          ansible.builtin.uri:
            url: "{{ vault_addr }}/v1/secret/data/terminus/agent/openclaw/api_key"
            method: GET
            headers:
              X-Vault-Token: "{{ openclaw_login.json.auth.client_token }}"
            ca_path: "{{ vault_ca_cert_path }}"
            status_code: 200
          register: check2_result

        - name: "CHECK 2 — PASS"
          ansible.builtin.debug:
            msg: "openclaw READ secret/terminus/agent/openclaw/api_key — PASS (own namespace)"
          when: check2_result.status == 200

        - name: "CHECK 3 — OpenClaw cannot read infra/postgres (expected: 403 Forbidden)"
          ansible.builtin.uri:
            url: "{{ vault_addr }}/v1/secret/data/terminus/infra/postgres/app_role_password"
            method: GET
            headers:
              X-Vault-Token: "{{ openclaw_login.json.auth.client_token }}"
            ca_path: "{{ vault_ca_cert_path }}"
            status_code: 403
          register: check3_result

        - name: "CHECK 3 — PASS (403 confirmed, isolation enforced)"
          ansible.builtin.debug:
            msg: "openclaw READ secret/terminus/infra/postgres/app_role_password — PASS (403 expected, access denied as required)"
          when: check3_result.status == 403

        - name: "ISOLATION VERIFICATION: ALL CHECKS PASSED"
          ansible.builtin.debug:
            msg: "ISOLATION VERIFICATION: ALL CHECKS PASSED"

        - name: "Fail if any check did not meet expected status"
          ansible.builtin.fail:
            msg: "Isolation check failed — review task output above"
          when: >
            check1_result.status != 200 or
            check2_result.status != 200 or
            check3_result.status != 403
    ```

- [ ] **Task 2: Ensure inventory hosts.yml includes `vault_vm` host with vars** (AC: 1)
  - [ ] This is partially fulfilled by Story 4.1 (Ansible control structure). If 4.1 is not yet complete, create a minimal `ansible/inventory/hosts.yml` stub:
    ```yaml
    all:
      hosts:
        vault_vm:
          ansible_host: "{{ lookup('pipe', 'cd ../../ && tofu output -raw vm_ip') }}"
          vault_addr: "https://{{ ansible_host }}:8200"
          vault_ca_cert_path: "/etc/vault.d/tls/ca.crt"
    ```
  - [ ] If Story 4.1 (Ansible control structure) is being done concurrently, coordinate inventory structure to avoid duplication

- [ ] **Task 3: Test playbook with correct credentials (AC: 1, 2, 3, 5)** 
  - [ ] Run: `ansible-playbook ansible/playbooks/verify-isolation.yml`
  - [ ] Provide valid role_ids and secret_ids when prompted
  - [ ] Confirm output contains all three PASS messages and "ISOLATION VERIFICATION: ALL CHECKS PASSED"
  - [ ] Run a second time (idempotency): same output, no Vault state change

- [ ] **Task 4: Test failure case (AC: 4)**
  - [ ] Temporarily remove the openclaw policy's `deny` (or add a temporary over-grant to openclaw to allow reading infra/)
  - [ ] Run the playbook again
  - [ ] Confirm non-zero exit code (Ansible fail task triggers)
  - [ ] Revert the temporary change and re-run to confirm PASS resumes

- [ ] **Task 5: Verify audit log records all playbook operations** (AC: 6)
  - [ ] After successful playbook run: `tail -20 /var/log/vault/audit.log | grep -E 'opentofu|openclaw'`
  - [ ] Confirm read successes (opentofu check 1, openclaw check 2) and the denied read (openclaw check 3) all appear

## Dev Notes

### Architecture Constraints

- **D8 — verify-isolation.yml is a first-class operational artifact:** The architecture decision D8 establishes `ansible/playbooks/verify-isolation.yml` as a required operational artifact. It is not an optional test — it is the canonical isolation verification mechanism and should be run:
  - After initial `vault-config` module apply
  - After any policy change to AppRole roles
  - After any `tofu apply` that touches `vault_policy` resources
  - As a post-deploy health check in CI/CD
  [Source: docs/terminus/infra/secrets/architecture.md#Additional Decisions — D8]

- **Concurrent development:** This story and Stories 3.3–3.5 are concurrent. The playbook should be built incrementally: start with Check 1 (opentofu infra read) as part of Story 3.3, add Check 2 (openclaw own-namespace read) with Story 3.5, and add Check 3 (openclaw cross-namespace denied) also as part of 3.5. By the time all three policy stories are done, all three checks should exist.

- **FR17 — all ops logged:** The playbook generates Vault audit entries for both successful reads and denied reads. This is by design and confirms the audit backend is working correctly during verification. Document this dual purpose (isolation check + audit verification) in the playbook comments.

- **vars_prompt vs environment variables:** In a CI/CD environment, `vars_prompt` should be replaced by `--extra-vars` from vault or environment injection. The playbook as written is suitable for manual operator use. Add a comment explaining CI/CD invocation pattern.

- **Secrets in ansible:** The `private: true` on secret_id prompts prevents output to terminal. In CI, consider using Ansible Vault (`ansible-vault`) to store test credentials or inject them via environment variables.

### Project Structure Notes

- Creates `ansible/playbooks/verify-isolation.yml` (new file)
- May create `ansible/inventory/hosts.yml` stub if Story 4.1 is not yet in place (coordinate with 4.1 developer)

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 3.7 (including Sprint Planning Note)]
- [Source: docs/terminus/infra/secrets/architecture.md#Additional Decisions — D8]
- [Source: docs/terminus/infra/secrets/architecture.md#Additional Requirements — FR17]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
