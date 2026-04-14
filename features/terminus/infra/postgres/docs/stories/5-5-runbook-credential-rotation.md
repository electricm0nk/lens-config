# Story 5.5: Runbook: Credential rotation

Status: ready-for-dev

## Story

As an operator, I want a runbook for revoking and reissuing a service role credential, so that FR9 can be executed on demand without prior context.

## Acceptance Criteria

1. Covers: trigger rotation in OpenTofu → `tofu apply` → verify old credential rejected → verify new credential works
2. Stored at `docs/terminus/infra/postgres/runbooks/05-credential-rotation.md`

## Tasks / Subtasks

- [ ] Outline runbook sections (AC: 1)
  - [ ] Section 1: Prerequisites — Identify target database; confirm SOPS state decrypted
  - [ ] Section 2: Trigger rotation — `tofu taint module.{db_name}.random_password.role_password`
  - [ ] Section 3: Apply — `tofu plan` (confirms 2 resources: password, vault secret); `tofu apply`
  - [ ] Section 4: Verify old credential rejected — `psql` with old password → confirm `FATAL: password authentication failed`
  - [ ] Section 5: Verify new credential works — `vault kv get secret/terminus/dev/postgres/{db_name}` → `psql` with new password
  - [ ] Section 6: Commit encrypted state — `sops -e terraform.tfstate > terraform.tfstate.enc && git commit`
  - [ ] Section 7: Notify consuming service — note: any service using the old credential must be updated
- [ ] Provide worked example using `acceptance_test_db` (AC: 1)
  - [ ] Exact taint command; exact apply command; expected `psql` error after rotation
- [ ] Commit runbook to `terminus.infra` (AC: 2)

## Dev Notes

### Architecture Context

Credential rotation uses `tofu taint + apply` to regenerate the `random_password` resource, update the PostgreSQL role password, and write the new credential bundle to Vault. The old password is immediately invalid after `ALTER ROLE ... PASSWORD` executes.

- Rotation mechanism: `tofu taint module.{db_name}.random_password.role_password`
- Vault path updated atomically with the PostgreSQL role password
- Active database connections using the old password will be terminated at connection renewal
- Important: consuming services must be restarted or reconfigured with the new credential

### Responsibility Split

**This story does:**
- Authors the credential rotation runbook

**This story does NOT:**
- Modify the `postgres-databases` module
- Rotate credentials for any specific database outside the worked example

### Directory Layout

```
terminus.infra/
  docs/
    terminus/
      infra/
        postgres/
          runbooks/
            05-credential-rotation.md    # CREATE
```

### Testing Standards

- Runbook walkthrough: follow steps for `acceptance_test_db` rotation; all steps execute without error
- Old password rejected post-rotation confirmed
- New Vault credential verified working
- SOPS state commit step included

### References

- epics.md: Story 5.5 Acceptance Criteria
- architecture.md: FR9 — Operator can revoke service role credential and issue new one without affecting other identities
- architecture.md: FR21 — Credential rotation documented as runbook

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `docs/terminus/infra/postgres/runbooks/05-credential-rotation.md` — CREATE
