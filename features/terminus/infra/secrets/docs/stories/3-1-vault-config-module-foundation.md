# Story 3.1: vault-config Module Foundation — Default-Deny and Audit Backend (FR10, FR17, FR18, FR20)

Status: ready-for-dev

## Story

As the operator,
I want the `vault-config/` OpenTofu module to configure the Vault audit backend and enforce its default-deny posture,
so that every Vault operation is logged and no path is accessible without explicit policy grant.

## Acceptance Criteria

1. **[Audit backend configured]** Given the vault-core module is applied (AppRole auth and KV v2 mount active), when `vault-config/` module is applied, then `vault audit list` includes a `file` backend writing to `/var/log/vault/audit.log`.

2. **[Audit log is structured JSON]** Given the audit log backend is active, when I run `head -1 /var/log/vault/audit.log | python3 -m json.tool`, then the command returns a valid JSON object (FR35 — structured, SIEM-compatible format).

3. **[Default-deny enforced (FR10)]** Given the default Vault policy, when I attempt `vault kv get secret/terminus/infra/postgres/app_role_password` using only the `default` policy token (no custom grants), then the response is HTTP 403 Forbidden — the default policy must not grant any access to `secret/` paths.

4. **[Audit entry appears within 30s (FR18)]** Given the audit log is active, when I perform any Vault operation (read, write, auth), then within 30 seconds an audit entry appears in `/var/log/vault/audit.log` with the path, operation, and requester identity.

5. **[Audit entry format correct]** Given a secret read operation, when I inspect the corresponding audit log entry, then `type` field is `response` and `auth.display_name` contains the identity of the caller.

6. **[Privileged ops queryable from audit log (FR20)]** Given the audit log is active and structured JSON, when I run `grep '"type":"response"' /var/log/vault/audit.log | jq 'select(.request.operation == "update" or .request.path | contains("auth/"))'`, then privileged operations (auth method changes, policy writes) are filterable from the log.

## Tasks / Subtasks

- [ ] **Task 1: Add `vault_audit.file` resource to `modules/vault-config/main.tf`** (AC: 1, 2)
  - [ ] Add:
    ```hcl
    resource "vault_audit" "file" {
      type = "file"
      options = {
        file_path = "/var/log/vault/audit.log"
      }
    }
    ```
  - [ ] Ensure `/var/log/vault/` directory exists on the VM with appropriate permissions (vault user writable) — this may need a `null_resource` with `remote-exec` to create the dir if not already created during Vault install (Story 1.5)
  - [ ] Resource name: `vault_audit.file`

- [ ] **Task 2: Verify default policy does not grant secret/ access (FR10)** (AC: 3)
  - [ ] Obtain a token with only the `default` policy: `vault token create -orphan -policy=default -ttl=1m`
  - [ ] Attempt read: `VAULT_TOKEN=<default-policy-token> vault kv get secret/terminus/infra/postgres/app_role_password`
  - [ ] Confirm 403 response
  - [ ] If the default policy grants unexpected access, add an explicit `path "secret/*" { capabilities = ["deny"] }` entry — but the Vault default policy should NOT grant this by default; check and document

- [ ] **Task 3: Add audit log directory creation to vault-core provisioner** (AC: 1)
  - [ ] If `/var/log/vault/` was not created in Story 1.5, add it to the Vault installation provisioner:
    ```bash
    mkdir -p /var/log/vault
    chown vault:vault /var/log/vault
    chmod 750 /var/log/vault
    ```
  - [ ] This may require amending a null_resource in `modules/vault-core/main.tf`

- [ ] **Task 4: Verify audit entry timing (FR18)** (AC: 4, 5)
  - [ ] Perform a Vault operation (e.g., `vault kv get secret/terminus/infra/postgres/app_role_password`)
  - [ ] Check `/var/log/vault/audit.log` within 30 seconds
  - [ ] Confirm: `head -1 /var/log/vault/audit.log | python3 -m json.tool` returns valid JSON
  - [ ] Confirm: `jq '.type' <<< "$(head -1 /var/log/vault/audit.log)"` returns `"response"`
  - [ ] Confirm: `jq '.auth.display_name' <<< "$(head -1 /var/log/vault/audit.log)"` shows the caller identity

- [ ] **Task 5: Document audit log querying in runbook** (AC: 6)
  - [ ] Add `## Audit Log Queries` section to `docs/runbooks/day0-bootstrap.md` or a new `docs/runbooks/operations.md`
  - [ ] Include example queries: find privileged ops, find denied operations, find operations by requester
  - [ ] Note the Vault 1.21.4 audit log JSON schema baseline (architecture requirement: schema version must be documented)

## Dev Notes

### Architecture Constraints

- **D7 — Audit log location:** `/var/log/vault/audit.log` is the canonical log path per architecture decision D7. logrotate (Story 4.2) will be configured to rotate this file. The path must be consistent across all stories.
  [Source: docs/terminus/infra/secrets/architecture.md#Operational Infrastructure — D7]

- **FR17 — Every operation logged:** Once enabled, the Vault audit backend logs every API request and response. The `file` backend is synchronous — Vault BLOCKS requests if the audit log cannot be written (disk full, permissions error). This is the correct behavior for a security-critical audit trail; document it.

- **Vault audit log JSON schema baseline (architecture requirement):** The architecture document states: "Vault 1.21.4 JSON audit log schema documented as baseline. Schema version noted in architecture; upgrade notes required for Vault version bumps." Add a note to the operations runbook: "Vault audit log schema version: Vault 1.21.4. Key fields: `type`, `time`, `auth.display_name`, `request.path`, `request.operation`, `response.data`. Schema drift on Vault upgrades must be reviewed before upgrading."
  [Source: docs/terminus/infra/secrets/architecture.md#Operational Infrastructure — D7]

- **Default-deny is Vault's default behavior:** Vault's built-in default policy grants access to `auth/token/lookup-self` and a few other self-service paths, but NOT to `secret/`. The default policy does not need to be modified — but it must be verified. If any other policy (e.g., `root`) is present on default tokens, that would be a misconfiguration.

- **vault-config authentication:** As noted in Story 2.4, after root token revocation vault-config must authenticate via a non-root mechanism. The `vault_audit` resource requires an authenticated Vault client. Ensure the vault-config module's Vault provider is configured with an appropriate token.

### Project Structure Notes

- Modifies `modules/vault-config/main.tf` (adds `vault_audit.file` resource)
- May modify `modules/vault-core/main.tf` (adds audit log directory creation to Vault install step)
- Creates or updates `docs/runbooks/` with audit log documentation

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 3.1]
- [Source: docs/terminus/infra/secrets/architecture.md#Operational Infrastructure — D7]
- [Source: docs/terminus/infra/secrets/architecture.md#Additional Requirements — FR17/FR10]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
