# Story 3.6: Secret-ID Revocation and Wrapped Token Delivery (FR8, FR9)

Status: ready-for-dev

## Story

As the operator,
I want to be able to revoke individual AppRole secret-IDs by accessor without affecting other roles or other secret-IDs of the same role, and to deliver new secret-IDs via single-use wrapped tokens,
so that compromised credentials can be surgically invalidated and new credentials cannot be intercepted in transit.

## Acceptance Criteria

1. **[Secret-ID revocation by accessor]** Given a k3s AppRole has two active secret-IDs (accessor A and accessor B), when I run `vault write auth/approle/role/k3s/secret-id-accessor/destroy accessor=<accessor-A>`, then accessor A is destroyed and the secret-id associated with it can no longer authenticate.

2. **[Other secret-IDs unaffected (FR8)]** Given secret-ID accessor A is destroyed (per AC1), when I attempt an AppRole login using the secret-id associated with accessor B, then the login succeeds — accessor A's destruction is surgical and does not affect other credentials.

3. **[Other AppRole roles unaffected]** Given the k3s secret-ID is revoked (AC1), when I log in using the opentofu AppRole or openclaw AppRole with their own valid secret-IDs, then those logins succeed — revocation is per-role, per-accessor.

4. **[Wrapped token delivery — single-use (FR9)]** Given an operator generates a new secret-id via `vault write -wrap-ttl=60s -f auth/approle/role/k3s/secret-id`, then a wrapping token (not the actual secret-id) is returned. When the wrapping token is unwrapped exactly once with `vault unwrap <wrapping_token>`, the actual secret-id is returned with HTTP 200.

5. **[Wrapped token is single-use]** Given a wrapping token was already unwrapped once (per AC4), when I attempt to unwrap the same wrapping token a second time, then the request fails with an error (the wrapping token is invalidated after first use).

6. **[Accessor list is queryable]** Given active secret-IDs exist for an AppRole, when I run `vault list auth/approle/role/k3s/secret-id-accessor`, then a list of accessors is returned — confirming that accessor management is queryable without exposing the actual secret-ids.

## Tasks / Subtasks

- [ ] **Task 1: Create two secret-IDs for k3s and record accessors** (AC: 1, 2)
  - [ ] Generate secret-id #1: `vault write -f auth/approle/role/k3s/secret-id`
    - Record the `secret_id_accessor` from the response → assign to variable `ACCESSOR_A`
    - Record the `secret_id` → assign to variable `SECRET_ID_A`
  - [ ] Generate secret-id #2: `vault write -f auth/approle/role/k3s/secret-id`
    - Record the `secret_id_accessor` → assign to variable `ACCESSOR_B`
    - Record the `secret_id` → assign to variable `SECRET_ID_B`
  - [ ] Confirm both are listed: `vault list auth/approle/role/k3s/secret-id-accessor` → shows both accessors (AC: 6)

- [ ] **Task 2: Verify AppRole login works with both secret-IDs before revocation** (AC: 2)
  - [ ] Login test A: `vault write auth/approle/login role_id=<k3s_role_id> secret_id=$SECRET_ID_A` → confirm success (HTTP 200, token returned)
  - [ ] Login test B: `vault write auth/approle/login role_id=<k3s_role_id> secret_id=$SECRET_ID_B` → confirm success
  - [ ] Note: if using single-use secret-ids, re-generate before this step; alternatively ensure `secret_id_num_uses = 0` (unlimited) for this test

- [ ] **Task 3: Revoke accessor A and verify revocation** (AC: 1)
  - [ ] Run: `vault write auth/approle/role/k3s/secret-id-accessor/destroy accessor=$ACCESSOR_A`
  - [ ] Confirm the command exits 0 (no error)
  - [ ] Verify accessor A is gone: `vault list auth/approle/role/k3s/secret-id-accessor` → `ACCESSOR_A` no longer listed

- [ ] **Task 4: Attempt login with revoked secret-id — verify failure** (AC: 1)
  - [ ] Attempt: `vault write auth/approle/login role_id=<k3s_role_id> secret_id=$SECRET_ID_A`
  - [ ] Confirm the response is HTTP 400 or 403 with "invalid secret id" or equivalent error

- [ ] **Task 5: Verify secret-id B still works (surgical revocation)** (AC: 2)
  - [ ] Attempt: `vault write auth/approle/login role_id=<k3s_role_id> secret_id=$SECRET_ID_B`
  - [ ] Confirm success — accessor B's secret-id is unaffected

- [ ] **Task 6: Verify other AppRole roles unaffected** (AC: 3)
  - [ ] With opentofu AppRole: generate a secret-id, attempt login → confirm success
  - [ ] With openclaw AppRole: generate a secret-id, attempt login → confirm success
  - [ ] Both succeed independently of k3s revocation

- [ ] **Task 7: Demonstrate wrapped token delivery** (AC: 4, 5)
  - [ ] Generate wrapped secret-id (60s TTL): `vault write -wrap-ttl=60s -f auth/approle/role/k3s/secret-id`
    - Record the `wrapping_token` from the response
  - [ ] Unwrap the first time: `vault unwrap <wrapping_token>` → confirm `secret_id` returned in JSON output (HTTP 200)
  - [ ] Attempt to unwrap a second time: `vault unwrap <wrapping_token>` → confirm error (wrapping token already used / expired)
  - [ ] Note: if wrap TTL expires before second attempt, regenerate and try both unwraps in quick succession

- [ ] **Task 8: Document revocation and wrapped delivery procedures** (no separate AC — extends AC4/AC5 validation)
  - [ ] Update `docs/runbooks/workload-onboarding.md` with section: `## Secret-ID Rotation and Revocation`
  - [ ] Include commands for: listing accessors, revoking by accessor, generating wrapped secret-ids, unwrapping
  - [ ] Reference FR8 (surgical revocation) and FR9 (single-use wrapped delivery)

## Dev Notes

### Architecture Constraints

- **FR8 — Surgical secret-id revocation:** Revocation must target a specific accessor, not the entire role. `vault write auth/approle/role/<role>/secret-id/destroy secret_id=<raw>` destroys by raw value (useful if you have the secret_id). `vault write auth/approle/role/<role>/secret-id-accessor/destroy accessor=<accessor>` destroys by accessor (useful when you only logged the accessor, not the raw secret_id — which is the correct operational practice: never log raw secret_ids). Use the accessor-based form.

- **FR9 — Wrapped token delivery:** The Response Wrapping feature (Cubbyhole) stores the actual secret-id in a Cubbyhole token accessible only once. The wrapping token is exchanged for the actual secret (unwrapped) exactly once. The operator delivers the wrapping token to the consumer (simulating a CI/CD pipeline or agent); the consumer unwraps it to get the actual secret-id. This pattern prevents MITM interception of the raw secret-id.

- **wrap-ttl operationalization:** `60s` is a safe wrap-ttl for testing. Production wrap-ttl should be long enough for the delivery channel latency but as short as possible (e.g., `5m` for automated CI pipeline, `30m` for manual delivery). Document the wrap-ttl policy in the runbook.

- **secret_id_num_uses:** For production AppRole roles, consider `secret_id_num_uses = 1` (single-use secret-id) for maximum security. This forces re-issue on each AppRole login and works well when paired with wrapped delivery. The current Story 3.3/3.4/3.5 configuration sets `secret_id_num_uses = 0` (unlimited). This story does NOT change that — it only demonstrates the revocation and wrapping mechanics. A follow-on security review may tighten this.

- **No Terraform resources:** This story is entirely operational procedure — no Terraform/OpenTofu resources are added. The vault-config module is unchanged. This is a verification and runbook story.

### Project Structure Notes

- No new Terraform files
- Modifies `docs/runbooks/workload-onboarding.md` (adds Secret-ID Rotation and Revocation section)

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 3.6]
- [Source: docs/terminus/infra/secrets/architecture.md#AppRole Vault Integration — FR8/FR9]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
