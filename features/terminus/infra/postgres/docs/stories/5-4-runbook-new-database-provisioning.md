# Story 5.4: Runbook: New database provisioning

Status: ready-for-dev

## Story

As an operator, I want a runbook for provisioning a new database and retrieving its credentials from Vault, so that consuming teams can onboard a new database without involving the original implementors.

## Acceptance Criteria

1. Covers: add database declaration to OpenTofu → `tofu apply` → retrieve credentials via `vault kv get` → verify connection
2. Stored at `docs/terminus/infra/postgres/runbooks/04-new-database.md`

## Tasks / Subtasks

- [ ] Outline runbook sections (AC: 1)
  - [ ] Section 1: Prerequisites — Confirm `tofu` and `vault` CLI available, dev environment SOPS state decrypted
  - [ ] Section 2: Declare new database — Add `module "my_service_db"` block to `tofu/environments/dev/main.tf`
  - [ ] Section 3: Plan and apply — `tofu plan` (review: 4 resources to add); `tofu apply`
  - [ ] Section 4: Retrieve credentials — `vault kv get secret/terminus/dev/postgres/my_service_db`
  - [ ] Section 5: Verify connection — `psql` with Vault credentials and `sslmode=require`
  - [ ] Section 6: Commit encrypted state — `sops -e terraform.tfstate > terraform.tfstate.enc && git commit`
- [ ] Provide worked example using `example_service_db` as the new database name (AC: 1, 2)
  - [ ] Show the exact `module` block to add to `main.tf`
  - [ ] Show expected `vault kv get` output format
  - [ ] Show expected `psql` connection string
- [ ] Commit runbook to `terminus.infra` (AC: 2)

## Dev Notes

### Architecture Context

The `postgres-databases` module is the automation primitive for all new database provisioning. The runbook documents how to invoke it. No direct VM access or psql superuser commands are needed — everything goes through OpenTofu.

- Module: `tofu/modules/postgres-databases/`
- New database declaration: add `module` block in `tofu/environments/dev/main.tf`
- Role name convention: `{db_name}_app`
- Vault path: `secret/terminus/dev/postgres/{db_name}`
- Post-apply: encrypt and commit new state file

### Responsibility Split

**This story does:**
- Authors the new database provisioning runbook

**This story does NOT:**
- Provision specific consuming service databases (those are their own initiatives)
- Modify the `postgres-databases` module

### Directory Layout

```
terminus.infra/
  docs/
    terminus/
      infra/
        postgres/
          runbooks/
            04-new-database.md    # CREATE
```

### Testing Standards

- Runbook walkthrough: a second operator follows it to provision `example_service_db` successfully
- All module block parameters shown with correct types and values
- Vault credential retrieval command produces expected 6-field output
- SOPS state commit step confirmed included

### References

- epics.md: Story 5.4 Acceptance Criteria
- architecture.md: FR3 — Provision database + role via automation; FR4 — Credentials to Vault KV v2
- architecture.md: FR21 — All provisioning operations documented as runbooks
- architecture.md: NFR5 — No plaintext credentials outside Vault

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `docs/terminus/infra/postgres/runbooks/04-new-database.md` — CREATE
