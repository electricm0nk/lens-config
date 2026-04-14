# Story 4.1: Implement `postgres-databases` OpenTofu module

Status: ready-for-dev

## Story

As an operator, I want a reusable `postgres-databases` OpenTofu module that provisions a database, scoped role, grants, and Vault credential write in a single `tofu apply`, so that FR3–FR6 are covered by a repeatable, idempotent pattern.

## Acceptance Criteria

1. Module accepts `db_name` and `environment` as inputs
2. Creates `postgresql_database`, `postgresql_role` (login, no superuser), `postgresql_grant` (CONNECT + CREATE on own DB only)
3. Writes full credential bundle to `secret/terminus/{env}/postgres/{dbname}` in Vault KV v2 (username, password, host, port, db_name, ssl_mode)
4. `tofu apply` is idempotent — re-run produces no changes if state matches

## Tasks / Subtasks

- [ ] Create `tofu/modules/postgres-databases/` directory and module files (AC: 1, 2, 3, 4)
- [ ] Write `variables.tf` (AC: 1)
  - [ ] `db_name` — string, the database name
  - [ ] `environment` — string, e.g., `dev`
  - [ ] `pg_host` — string, PostgreSQL host or Consul DNS name
  - [ ] `pg_port` — number, default `5432`
  - [ ] `vault_mount` — string, default `secret`
  - [ ] `admin_username`, `admin_password` — break-glass admin credentials for provider auth
- [ ] Write `main.tf` (AC: 2, 3)
  - [ ] `postgresql_database` resource: `name = var.db_name`, encoding UTF8
  - [ ] `postgresql_role` resource: `name = "${var.db_name}_app"`, `login = true`, `superuser = false`, random password via `random_password`
  - [ ] `postgresql_grant` resource: CONNECT privilege on database + CREATE on own schema
  - [ ] `vault_kv_secret_v2` resource: path `${var.vault_mount}/terminus/${var.environment}/postgres/${var.db_name}`, data: username, password, host, port, db_name, ssl_mode=require
- [ ] Write `outputs.tf` (AC: 1)
  - [ ] `db_name`, `role_name`, `vault_path` outputs
- [ ] Write `versions.tf` (AC: 4)
  - [ ] Pin `cyrilgdn/postgresql` and `hashicorp/vault` provider versions
- [ ] Wire `acceptance_test_db` instance into `tofu/environments/dev/main.tf` (AC: 1, 4)
  - [ ] Add `module "acceptance_test_db"` calling `modules/postgres-databases`
- [ ] Run `tofu apply` and verify idempotency with second apply (AC: 4)

## Dev Notes

### Architecture Context

The `postgres-databases` module is the key automation primitive for FR3–FR6. It uses two OpenTofu providers:
- `cyrilgdn/postgresql` — database, role, and grant management
- `hashicorp/vault` — Vault KV v2 credential write-back

The provider block for `cyrilgdn/postgresql` must connect to the running PostgreSQL cluster using the break-glass admin credential. This requires the Vault admin credential to be available at `tofu apply` time.

- Vault path: `secret/terminus/{env}/postgres/{dbname}`
- Vault schema: `{username, password, host, port, db_name, ssl_mode}`
- Role naming convention: `{db_name}_app`
- `ssl_mode` stored as `require` always (NFR4, FR8)

### Responsibility Split

**This story does:**
- Creates the `postgres-databases` module
- Provisions `acceptance_test_db` as the first module instance (Story 4.2 builds on this)

**This story does NOT:**
- Verify Vault credentials end-to-end (Story 4.2)
- Verify role isolation (Story 4.3)
- Test credential rotation (Story 4.4)

### Directory Layout

```
terminus.infra/
  tofu/
    modules/
      postgres-databases/
        main.tf          # CREATE — database, role, grant, vault write resources
        variables.tf     # CREATE — db_name, environment, pg_host, etc.
        outputs.tf       # CREATE — db_name, role_name, vault_path
        versions.tf      # CREATE — provider version pins
    environments/
      dev/
        main.tf          # MODIFY — add module "acceptance_test_db" block
```

### Testing Standards

- `tofu plan` shows only `acceptance_test_db` resources (no drift on cluster VMs or backup)
- `tofu apply` exits 0
- Second `tofu apply` immediately after first: "No changes, infrastructure is up-to-date."
- `vault kv get secret/terminus/dev/postgres/acceptance_test_db` returns full credential bundle with all 6 fields populated

### References

- epics.md: Story 4.1 Acceptance Criteria
- architecture.md: FR3 — Provision database + role via automation; FR4 — Credentials to Vault KV v2
- architecture.md: FR5 — Retrieve credentials from Vault; FR6 — Per-database scoped role, no superuser
- architecture.md: "IaC DB/role provider: OpenTofu cyrilgdn/postgresql; IaC Vault provider: hashicorp/vault"
- architecture.md: "Vault paths: secret/terminus/{env}/postgres/{dbname}"

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `tofu/modules/postgres-databases/main.tf` — CREATE
- `tofu/modules/postgres-databases/variables.tf` — CREATE
- `tofu/modules/postgres-databases/outputs.tf` — CREATE
- `tofu/modules/postgres-databases/versions.tf` — CREATE
- `tofu/environments/dev/main.tf` — MODIFY (add acceptance_test_db module block)
