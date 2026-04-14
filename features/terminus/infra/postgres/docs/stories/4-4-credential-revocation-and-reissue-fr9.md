# Story 4.4: Credential revocation and reissue (FR9)

Status: ready-for-dev

## Story

As an operator, I want to revoke the `acceptance_test_db_app` credential and issue a new one without affecting other database identities, so that FR9 is verified (acceptance Test 4).

## Acceptance Criteria

1. Old credential (`psql` connect) works before rotation
2. `tofu apply` with credential rotation writes new credential to Vault
3. Old credential is rejected by PostgreSQL after rotation (`FATAL: password authentication failed`)
4. New credential retrieved from Vault connects successfully
5. Other database credentials are unaffected

## Tasks / Subtasks

- [ ] Add credential rotation support to `postgres-databases` module (AC: 2)
  - [ ] Add `rotate_password` boolean variable (or use `lifecycle { create_before_destroy }` on `random_password`)
  - [ ] On `tofu apply`: `random_password` resource tainted or new resource triggers role password update
  - [ ] `postgresql_role` resource picks up new password; `vault_kv_secret_v2` writes new bundle
- [ ] Pre-rotation: confirm old credential connects (AC: 1)
  - [ ] `psql "... user=acceptance_test_db_app sslmode=require"` with current Vault password — success
- [ ] Execute rotation: taint `random_password` or trigger rotation variable (AC: 2)
  - [ ] `tofu taint module.acceptance_test_db.random_password.role_password`
  - [ ] `tofu apply` — updates role password in PostgreSQL and writes new credential to Vault
- [ ] Verify old credential rejected (AC: 3)
  - [ ] `psql` attempt with old password — confirm `FATAL: password authentication failed`
- [ ] Verify new credential works (AC: 4)
  - [ ] `vault kv get secret/terminus/{env}/postgres/acceptance_test_db` — retrieve new password
  - [ ] `psql` with new Vault credential — success
- [ ] Verify no other database credentials affected (AC: 5)
  - [ ] If any other module instances exist, verify their Vault credentials unchanged after rotation

## Dev Notes

### Architecture Context

Credential rotation works by regenerating the `random_password` resource for the target role via `tofu taint + apply`. The `postgresql_role` resource updates the password in PostgreSQL, and the `vault_kv_secret_v2` resource writes the new credential bundle. No other role's password or Vault secret is touched.

- Vault path: `secret/terminus/{env}/postgres/{dbname}`
- PostgreSQL role: `{dbname}_app`
- Rotation mechanism: `tofu taint module.{db}.random_password.role_password && tofu apply`
- Old sessions using the old password will be disconnected after `ALTER ROLE ... PASSWORD`

### Responsibility Split

**This story does:**
- Adds rotation support to `postgres-databases` module (if not already sufficient from Story 4.1)
- Verifies full rotation cycle: old works → rotate → old rejected → new works

**This story does NOT:**
- Create the original credential (Story 4.1, 4.2)
- Test role isolation (Story 4.3)
- Run the full acceptance test suite (Story 4.5)

### Directory Layout

```
terminus.infra/
  tofu/
    modules/
      postgres-databases/
        main.tf    # MODIFY — ensure random_password lifecycle supports rotation via taint
```

### Testing Standards

- Pre-rotation `psql` with old password: exit 0
- `tofu apply` after taint: applies new password resource, no errors
- Post-rotation `psql` with old password: `FATAL: password authentication failed`
- Post-rotation `psql` with new Vault credential: exit 0
- `vault kv get` before and after: version number increments, new password field differs

### References

- epics.md: Story 4.4 Acceptance Criteria
- architecture.md: FR9 — Operator can revoke a service role credential and issue a new one without affecting other database identities
- architecture.md: NFR5 — Database passwords never stored in plaintext outside Vault

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `tofu/modules/postgres-databases/main.tf` — MODIFY (rotation support if needed)
