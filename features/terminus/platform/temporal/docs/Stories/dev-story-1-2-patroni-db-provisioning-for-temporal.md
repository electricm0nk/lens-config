# Story 1.2: Patroni DB Provisioning for Temporal

Status: done

## Story

As a platform developer,
I want PostgreSQL databases and credentials provisioned on the Patroni cluster before the Temporal Helm chart is applied,
so that the Temporal schema migration Jobs can connect to the expected databases and complete successfully on first sync.

## Acceptance Criteria

1. SQL provisioning script committed to `docs/scripts/db-provision-temporal.sql` in `terminus.platform`
2. Script creates `temporal` database
3. Script creates `temporal_visibility` database
4. Script creates `temporal` DB user with password reference (no plaintext password in script)
5. Script grants `temporal` user full privileges on both databases
6. Vault secret at `secret/terminus/default/temporal/db-password` created with the generated password
7. Vault secret at `secret/terminus/default/temporal/visibility-db-password` created (same or separate user — decision documented in script header)
8. Script is idempotent (uses `CREATE DATABASE IF NOT EXISTS`, `CREATE USER IF NOT EXISTS`)
9. Runbook for running the script documented in script header comments: `psql -h 10.0.0.56 -U admin -f db-provision-temporal.sql`

## Tasks / Subtasks

- [ ] Task 1: Write and commit SQL provisioning script (AC: 1–5, 8, 9)
  - [ ] Create `docs/scripts/db-provision-temporal.sql` in `terminus.platform`
  - [ ] Add header comment block: purpose, Patroni host, admin user, idempotency guarantee
  - [ ] Add runbook comment: `psql -h 10.0.0.56 -U admin -f db-provision-temporal.sql`
  - [ ] Implement `CREATE DATABASE IF NOT EXISTS temporal`
  - [ ] Implement `CREATE DATABASE IF NOT EXISTS temporal_visibility`
  - [ ] Implement `CREATE USER IF NOT EXISTS temporal WITH ENCRYPTED PASSWORD '{from-vault}'`
  - [ ] Document that password is sourced from Vault at `secret/terminus/default/patroni/admin-password` — never in script
  - [ ] `GRANT ALL PRIVILEGES ON DATABASE temporal TO temporal;`
  - [ ] `GRANT ALL PRIVILEGES ON DATABASE temporal_visibility TO temporal;`
  - [ ] Document decision: `temporal_visibility` uses the same `temporal` user (two Vault paths for consistency)
  - [ ] Commit: `feat(infra): add Patroni DB provisioning script for Temporal`

- [ ] Task 2: Provision Vault secrets (AC: 6, 7)
  - [ ] Generate a strong password for the `temporal` DB user
  - [ ] Write to Vault: `vault kv put secret/terminus/default/temporal/db-password password=<generated>`
  - [ ] Write to Vault: `vault kv put secret/terminus/default/temporal/visibility-db-password password=<same>`
  - [ ] Verify both secrets are readable: `vault kv get secret/terminus/default/temporal/db-password`

- [ ] Task 3: Execute provisioning script against Patroni (AC: all)
  - [ ] Retrieve admin password: `vault kv get -field=password secret/terminus/default/patroni/admin-password`
  - [ ] Run: `psql -h 10.0.0.56 -U admin -f docs/scripts/db-provision-temporal.sql`
  - [ ] Verify: `psql -h 10.0.0.56 -U admin -c "\l"` shows `temporal` and `temporal_visibility` databases
  - [ ] Verify: `psql -h 10.0.0.56 -U admin -c "\du"` shows `temporal` user

## Dev Notes

### Target Repo Path
```
TargetProjects/terminus/platform/terminus.platform/
```

### Patroni Connection Details
- Primary host: `10.0.0.56:5432`
- Admin credentials: `vault kv get secret/terminus/default/patroni/admin-password` (established convention)

### Vault Secret Layout
Both secrets use the `temporal` user password — two separate Vault paths for future flexibility:
- `secret/terminus/default/temporal/db-password` → `temporal` DB user password (used by `temporal` database)
- `secret/terminus/default/temporal/visibility-db-password` → same password (used by `temporal_visibility` database)

---

## Architecture Alignment Note (Updated: 2026-04-06)

> **⚠️ DRIFT: Vault path uses `default/temporal/` but the architecture hierarchy assigns temporal credentials to `dev/temporal/`.**
>
> **What was built:** Vault secrets written to `secret/terminus/default/temporal/db-password` and `secret/terminus/default/temporal/visibility-db-password`.
>
> **Architecture hierarchy (architecture.md — Vault Path Hierarchy):**
> - `secret/terminus/default/` is reserved for **shared cross-service resources only** — currently `ghcr/pull-token`
> - Temporal DB credentials belong at `secret/terminus/dev/temporal/db-password`
>
> **Impact:** Downstream stories (2.2 ESO ExternalSecrets) reference the same `default/temporal/` paths. If the secrets were written to `default/`, they will resolve at runtime, but they are in the wrong Vault namespace per the architecture.
>
> **Remediation:** Move the Vault secrets to `secret/terminus/dev/temporal/db-password` and `secret/terminus/dev/temporal/visibility-db-password`, and update Story 2.2 ExternalSecret manifests to match. See Story 2.2 alignment note.

### Constitutional Override — Develop-First Integration
Implementation work lands on `develop` in `terminus.platform` (Terminus Art. 5 override active).

### Not In Scope
- ESO ExternalSecrets for these Vault keys (Story 2.2)
- Temporal Helm chart deployment (Story 2.3)
- Any Kubernetes manifests

## Pre-Implementation Constitution Gates

| Gate | Article | Result |
|------|---------|--------|
| Track declared | Org Art. 1 | ✅ PASS — `tech-change` |
| Architecture documented | Org Art. 3 | ✅ PASS — `docs/terminus/platform/temporal/architecture.md` |
| No confidential data | Org Art. 4 | ✅ PASS — no plaintext credentials in script or commit |
| Git discipline | Org Art. 5 | ✅ PASS — develop-first integration authorized by terminus Art. 5 |
| Security first | Org Art. 9 | ✅ PASS — all credentials via Vault, no plaintext |
| Repo as source of truth | Org Art. 10 | ✅ PASS — all context from committed docs |
| Develop-first integration | Terminus Art. 5 | ✅ OVERRIDE ACTIVE — land implementation work on `develop` in `terminus.platform` |

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (copilot)

### Debug Log References

### Completion Notes List

### File List
