# Story 1.1: Provision Semaphore Postgres Database and Role via OpenTofu

Status: ready-for-dev

## Story

As a platform operator,
I want a dedicated `semaphore` Postgres database and role provisioned in the Terminus Patroni cluster,
So that Semaphore UI has an isolated database with appropriate access credentials.

## Acceptance Criteria

1. **Given** the Patroni Postgres HA cluster is operational at `10.0.0.56:5432`
   **When** `tofu apply` runs in `tofu/environments/postgres/semaphore/` with the DB password variable sourced from Vault
   **Then** database `semaphore` exists in the Patroni cluster
2. Role `semaphore` exists with `GRANT ALL` on the `semaphore` database
3. Tofu state is stored in Consul at key `tofu/postgres/semaphore`
4. No credentials appear in any committed Tofu file (password passed via variable at apply time)
5. `psql -h 10.0.0.56 -U semaphore semaphore` connects successfully

## Tasks

- [ ] Task 1: Create `tofu/environments/postgres/semaphore/` in `terminus.infra`
  - [ ] Create `backend.tf` — Consul state backend, key `tofu/postgres/semaphore` (match pattern from existing `postgres/dev/` env)
  - [ ] Create `variables.tf` — declare `db_password` variable (type `string`, sensitive = true)
  - [ ] Create `main.tf` — call the `postgres-databases` module with `db = "semaphore"`, `role = "semaphore"`, `password = var.db_password`
- [ ] Task 2: Run Tofu provisioning (coordinate with Story 1.2 — DB password must match Vault value)
  - [ ] `tofu init` — verify remote Consul state backend connects
  - [ ] `tofu plan -var="db_password=<value-from-vault>"` — review plan, confirm only DB + role creation
  - [ ] `tofu apply` — confirm database `semaphore` and role `semaphore` created
- [ ] Task 3: Verify database accessible
  - [ ] `psql -h 10.0.0.56 -U semaphore semaphore` — confirm connection succeeds (password from Vault)
  - [ ] `\l` in psql — confirm `semaphore` database listed

## Dev Notes

### Cross-Story Dependency

**CRITICAL sequencing:**
- The DB password you use here in `tofu apply -var="db_password=..."` MUST be identical to the value stored in Vault at `secret/terminus/infra/semaphoreui` field `db_password` (which Story 1.2 configures).
- Recommended order: Seed Vault first (Story 1.2, Task 1), then run `tofu apply` here with the same password value.

### Repository Target

All Tofu files in this story are committed to `terminus.infra` repository (NOT the control/planning repo).

### Existing Pattern Reference

- Reference the existing `tofu/environments/postgres/dev/` environment for Consul backend config pattern.
- Do NOT modify anything in `postgres/dev/` — it manages the Patroni cluster itself.
- The `postgres-databases` module is the established pattern for user-facing DB provisioning.

### Homelab Constraint: Patroni Direct-IP

- `10.0.0.56` is the Patroni primary's direct IP. This is a known homelab constraint (no VIP abstraction in place). If the primary fails over, the IP may change.
- This is **accepted** for the current homelab scope. Document in implementation notes if primary IP changes.

### Architecture Reference

- [Source: docs/terminus/infra/semaphoreui/architecture.md — OpenTofu Environment section]
- [Source: docs/terminus/infra/semaphoreui/tech-decisions.md — TD-012 (Standalone Tofu env), TD-011 (Patroni backend)]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (GitHub Copilot)

### Debug Log References

### Completion Notes List

### File List

- `tofu/environments/postgres/semaphore/backend.tf` (new)
- `tofu/environments/postgres/semaphore/variables.tf` (new)
- `tofu/environments/postgres/semaphore/main.tf` (new)
