# Story 3.5: OpenClaw AppRole Identity and Isolation Policy — AI Agent Namespace (FR6, FR7, FR11)

Status: ready-for-dev

## Story

As the operator,
I want an openclaw-specific AppRole with a policy granting read access solely to `secret/terminus/agent/openclaw/*` and denying all infrastructure paths,
so that the AI agent workload is cryptographically isolated from infrastructure credentials.

## Acceptance Criteria

1. **[Policy file header matches convention]** Given the HCL policy file exists at `vault-config/policies/openclaw.hcl`, when I inspect it, then the first line reads exactly: `# Policy for openclaw AppRole — read-only on secret/terminus/agent/openclaw/* — deny all else`

2. **[Policy scoped to agent/openclaw/\* only (FR7)]** Given the openclaw policy is applied, when I read `vault policy read openclaw`, then it grants `capabilities = ["read", "list"]` on `secret/data/terminus/agent/openclaw/*` and `secret/metadata/terminus/agent/openclaw/*` — and contains no grants to any `infra/` path.

3. **[openclaw can read its own secrets]** Given an openclaw AppRole token, when I run `vault kv get secret/terminus/agent/openclaw/<any_key>`, then the operation succeeds on any populated path (or returns "no value found" without 403 if the path exists in the mount).

4. **[openclaw cannot read infra/postgres/\* (FR11 — lateral isolation)]** Given an openclaw AppRole token, when I attempt `vault kv get secret/terminus/infra/postgres/app_role_password`, then the response is HTTP 403 Forbidden.

5. **[openclaw cannot read infra/k3s/\* (FR11 — lateral isolation)]** Given an openclaw AppRole token, when I attempt `vault kv get secret/terminus/infra/k3s/<any_key>`, then the response is HTTP 403 Forbidden.

6. **[All denied attempts are audit-logged (FR17, FR20)]** Given the two denied access attempts from AC4 and AC5, when I inspect the Vault audit log, then entries for both denied operations appear with `error` containing "permission denied" and `auth.display_name` identifying the openclaw AppRole identity.

## Tasks / Subtasks

- [ ] **Task 1: Create `vault-config/policies/openclaw.hcl`** (AC: 1, 2)
  - [ ] Create file `modules/vault-config/policies/openclaw.hcl`:
    ```hcl
    # Policy for openclaw AppRole — read-only on secret/terminus/agent/openclaw/* — deny all else

    path "secret/data/terminus/agent/openclaw/*" {
      capabilities = ["read", "list"]
    }

    path "secret/metadata/terminus/agent/openclaw/*" {
      capabilities = ["read", "list"]
    }

    path "sys/leases/renew" {
      capabilities = ["update"]
    }
    ```
  - [ ] Note: the explicit deny of `secret/terminus/infra/*` is enforced by Vault default-deny (no grant = deny); explicit `deny` capabilities are optional but may be added for clarity if preferred

- [ ] **Task 2: Add `vault_policy.openclaw` resource** (AC: 2)
  - [ ] In `modules/vault-config/main.tf` (or `policies.tf`):
    ```hcl
    resource "vault_policy" "openclaw" {
      name   = "openclaw"
      policy = file("${path.module}/policies/openclaw.hcl")
    }
    ```

- [ ] **Task 3: Add `vault_approle_auth_backend_role.openclaw` resource** (AC: 3)
  - [ ] In `modules/vault-config/main.tf` (or `roles.tf`):
    ```hcl
    resource "vault_approle_auth_backend_role" "openclaw" {
      backend        = "approle"
      role_name      = "openclaw"
      token_policies = [vault_policy.openclaw.name]
      token_ttl      = 3600    # 1 hour (per Story 3.2)
      token_max_ttl  = 86400   # 24 hours (per Story 3.2)
      secret_id_num_uses = 0
    }
    ```

- [ ] **Task 4: Add `approle_role_id_openclaw` output** (AC: 3)
  - [ ] In `modules/vault-config/outputs.tf`:
    ```hcl
    output "approle_role_id_openclaw" {
      description = "Vault AppRole role_id for the openclaw AI agent identity"
      value       = vault_approle_auth_backend_role.openclaw.role_id
      sensitive   = false
    }
    ```

- [ ] **Task 5: Seed openclaw test secret and verify read** (AC: 3)
  - [ ] With an admin token, seed a test value:
    `vault kv put secret/terminus/agent/openclaw/api_key value="test-placeholder"`
  - [ ] Log in with openclaw AppRole: `vault write auth/approle/login role_id=<openclaw_role_id> secret_id=<generated>`
  - [ ] Verify read: `vault kv get secret/terminus/agent/openclaw/api_key` returns data
  - [ ] Verify token policies include `openclaw`: `vault token lookup | grep policies`

- [ ] **Task 6: Test lateral isolation — infra paths blocked** (AC: 4, 5)
  - [ ] With openclaw token: `vault kv get secret/terminus/infra/postgres/app_role_password` → confirm 403
  - [ ] With openclaw token: `vault kv get secret/terminus/infra/k3s/kubeconfig_admin` → confirm 403
  - [ ] Search audit log: `grep '"error"' /var/log/vault/audit.log | grep "openclaw" | tail -5` — confirm both entries present (AC: 6)

## Dev Notes

### Architecture Constraints

- **FR11 — AI Agent Isolation:** The openclaw workload represents the AI agent tooling in the terminus architecture. FR11 explicitly requires that AI agent credentials must be isolated from infrastructure credentials (postgres, k3s). The openclaw policy enforces this through Vault default-deny — no grant outside `agent/openclaw/*` means no access.

- **Explicit deny consideration:** Vault applies default-deny to all paths not explicitly granted. You do NOT need `deny` capabilities on infra paths — they are already denied. However, adding explicit deny stanzas (e.g., `path "secret/*" { capabilities = ["deny"] }` except the openclaw paths) creates a "belt-and-suspenders" defense against future policy merges. This is optional but worth discussing in the ops runbook.

- **Namespace design:** `agent/` is distinct from `infra/` in the secret path hierarchy, providing an additional organizational boundary. Future AI agents should be onboarded under `secret/terminus/agent/<agent-name>/` following the openclaw pattern.

- **Concurrent with 3.7:** The isolation verification suite (Story 3.7) is developed concurrently with Stories 3.3–3.5. As each AppRole identity is onboarded, a corresponding check should be added to `verify-isolation.yml`. The developer on this story should coordinate with Story 3.7 work to add the openclaw check to that playbook.

- **KV v2 path semantics:** Same as Stories 3.3 and 3.4 — always include both `secret/data/...` and `secret/metadata/...` paths.

### Project Structure Notes

- Creates `modules/vault-config/policies/openclaw.hcl` (new file)
- Modifies `modules/vault-config/main.tf` (adds vault_policy.openclaw and vault_approle_auth_backend_role.openclaw)
- Modifies `modules/vault-config/outputs.tf` (adds approle_role_id_openclaw output)

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 3.5]
- [Source: docs/terminus/infra/secrets/architecture.md#AppRole Vault Integration — FR6/FR7/FR11]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
