# Story 5.8: Runbook: Break-glass access

Status: ready-for-dev

## Story

As an operator, I want a runbook for retrieving the break-glass admin credential and accessing the cluster directly, so that emergency access is always recoverable from documentation alone.

## Acceptance Criteria

1. Covers: `vault kv get secret/terminus/{env}/postgres/admin` → direct `psql` connection → post-use steps (audit, rotate)
2. Stored at `docs/terminus/infra/postgres/runbooks/08-break-glass.md`

## Tasks / Subtasks

- [ ] Outline runbook sections (AC: 1)
  - [ ] Section 1: When to use break-glass — Criteria for using superuser access; note that all use is logged
  - [ ] Section 2: Retrieve credential — `vault kv get -field=password secret/terminus/{env}/postgres/admin`
  - [ ] Section 3: Direct psql connection — `psql "host=postgres-primary-vm user=<admin_user> dbname=postgres sslmode=require"`
  - [ ] Section 4: Verification — Confirm `SELECT current_user` returns superuser role
  - [ ] Section 5: Post-use mandatory steps — Note the access in a change record; rotate credential after use; verify rotation in Vault
  - [ ] Section 6: If Vault is unavailable — Age key recovery path for offline access (break-glass of the break-glass)
- [ ] Document when NOT to use break-glass (AC: 1)
  - [ ] Application workloads: use `acceptance_test_db_app` or respective service role
  - [ ] Routine database operations: use `postgres-databases` module
- [ ] Commit runbook to `terminus.infra` (AC: 2)

## Dev Notes

### Architecture Context

The break-glass admin credential is a superuser-level Vault credential for emergency operator access. It is stored at `secret/terminus/{env}/postgres/admin`. All connections using the superuser credential are captured in pgaudit JSON logs (Story 2.4). Access should be rare and always followed by credential rotation.

- Vault path: `secret/terminus/{env}/postgres/admin`
- Usage: emergency only — not for application workloads (NFR6)
- Post-use rotation: `tofu taint + apply` on the admin credential resource (or manual `ALTER ROLE ... PASSWORD`)
- If Vault down: age key recovery path should reference the SOPS-encrypted state which contains or references the credential bootstrap value

### Responsibility Split

**This story does:**
- Authors the break-glass access runbook

**This story does NOT:**
- Create the break-glass credential (established during cluster bootstrap in Story 2.2)
- Modify any infrastructure code

### Directory Layout

```
terminus.infra/
  docs/
    terminus/
      infra/
        postgres/
          runbooks/
            08-break-glass.md    # CREATE
```

### Testing Standards

- Vault credential retrieval command verified against actual dev cluster
- `psql` connection with break-glass credential succeeds
- Post-use steps clearly documented with no ambiguity
- "When NOT to use" section present and clear

### References

- epics.md: Story 5.8 Acceptance Criteria
- architecture.md: "Vault paths: secret/terminus/{env}/postgres/admin (break-glass)"
- architecture.md: NFR6 — No application workload uses superuser
- architecture.md: NFR12 — Audit logging captures all connections

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `docs/terminus/infra/postgres/runbooks/08-break-glass.md` — CREATE
