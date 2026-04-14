# Story 3.2: AppRole Token TTL Policy — All Roles Explicitly Bound (AR-S8 MEDIUM, NFR3)

Status: ready-for-dev

## Story

As the security architect,
I want every AppRole authentication backend role to have explicit `token_ttl` and `token_max_ttl` values,
so that no workload holds an unlimited or oversized Vault token and all tokens expire predictably within the 90-day compliance ceiling.

## Acceptance Criteria

1. **[Explicit TTL on every role (AR-S8)]** Given the vault-config module is applied, when I inspect `vault read auth/approle/role/<role>` for each AppRole role (`opentofu`, `k3s`, `openclaw`), then each role returns a `token_ttl` greater than 0 and a `token_max_ttl` greater than 0.

2. **[Max TTL ≤ 90 days (NFR3)]** Given each AppRole role is configured, when the `token_max_ttl` is read for each role, then its value is 7776000 seconds (90 days) or less for all roles.

3. **[Observed token TTL matches configured value]** Given an AppRole login produces a Vault token, when I run `vault token lookup <token>` on the resulting token, then the `ttl` field matches the `token_ttl` configured for that role (within 5 seconds tolerance due to login latency).

4. **[TTL rationale documented]** Given the TTL values are configured, when I read `docs/runbooks/workload-onboarding.md`, then it contains a `## Token TTL Policy` section documenting the TTL chosen per workload role and the rationale (short-lived preferred, longer TTL for automation roles with re-auth burden).

5. **[Default TTL removed or not relied upon]** Given the AppRole auth backend default settings, when I inspect the auth method config (`vault read auth/approle/tune`), then the `default_lease_ttl` and `max_lease_ttl` at the mount level are NOT the only TTL controls — each role must set its own explicit values (per AC1).

## Tasks / Subtasks

- [ ] **Task 1: Set token_ttl and token_max_ttl on opentofu AppRole role** (AC: 1, 2, 3)
  - [ ] In `modules/vault-config/main.tf` (or in a dedicated `roles.tf`), for `vault_approle_auth_backend_role.opentofu`, add or confirm:
    ```hcl
    token_ttl     = 3600   # 1 hour — OpenTofu apply short window
    token_max_ttl = 86400  # 24 hours — renewals limited to 1 day max
    ```
  - [ ] Rationale for opentofu: short TTL aligns with typical pipeline run time (~60 min); max_ttl at 24h caps token renewal chains

- [ ] **Task 2: Set token_ttl and token_max_ttl on k3s AppRole role** (AC: 1, 2, 3)
  - [ ] For `vault_approle_auth_backend_role.k3s`, add or confirm:
    ```hcl
    token_ttl     = 86400    # 24 hours — k3s service identity, rotates daily
    token_max_ttl = 2592000  # 30 days — max cap for long-running cluster
    ```
  - [ ] Rationale: k3s is a long-running service; 24h initial TTL with renewal to 30d max is appropriate for infra that cannot re-authenticate hourly

- [ ] **Task 3: Set token_ttl and token_max_ttl on openclaw AppRole role** (AC: 1, 2, 3)
  - [ ] For `vault_approle_auth_backend_role.openclaw`, add or confirm:
    ```hcl
    token_ttl     = 3600    # 1 hour — agent task-scoped
    token_max_ttl = 86400   # 24 hours — agent workload max
    ```
  - [ ] Rationale: AI agent workloads are typically short-lived task batches; 1h TTL enforces re-auth per session

- [ ] **Task 4: Verify 90-day ceiling (NFR3)** (AC: 2)
  - [ ] Confirm that all `token_max_ttl` values are ≤ 7776000 (90 × 86400)
  - [ ] Add a `check {}` or comment in the HCL:
    ```hcl
    # NFR3: token_max_ttl must not exceed 90 days (7776000s)
    # Current value: 2592000s (30 days) for k3s — within compliance ceiling
    ```

- [ ] **Task 5: Write token TTL policy documentation** (AC: 4)
  - [ ] Add `## Token TTL Policy` section to `docs/runbooks/workload-onboarding.md` (create stub if not yet present)
  - [ ] Include a table:
    | Role | token_ttl | token_max_ttl | Rationale |
    |---|---|---|---|
    | opentofu | 3600 (1h) | 86400 (24h) | Pipeline window, short-lived |
    | k3s | 86400 (24h) | 2592000 (30d) | Long-running service, daily re-auth |
    | openclaw | 3600 (1h) | 86400 (24h) | Agent task sessions, short-lived |
  - [ ] Note: all max_ttls are within NFR3 90-day ceiling

- [ ] **Task 6: Verify token lookup after login** (AC: 3)
  - [ ] After `vault-config` apply, log in with each AppRole: `vault write auth/approle/login role_id=... secret_id=...`
  - [ ] Run `vault token lookup <token>` for the returned token
  - [ ] Confirm `ttl` field in lookup output matches the configured `token_ttl`

## Dev Notes

### Architecture Constraints

- **AR-S8 — MEDIUM risk, no implicit defaults:** "All AppRole roles must have explicit token_ttl and token_max_ttl — no reliance on Vault auth mount defaults" (risk: unbounded tokens if mount defaults are changed). This story directly closes this risk item.
  [Source: docs/terminus/infra/secrets/architecture.md#Vault Provisioning Decisions]

- **NFR3 — 90-day maximum token lifetime:** "Max Vault token TTL is 90 days." Even though typical TTLs are much shorter, this constraint is the hard ceiling. Any vault_approle_auth_backend_role resource with `token_max_ttl` > 7776000 violates NFR3.

- **Token TTL vs Lease TTL:** Vault's auth token TTL and secret lease TTL are separate concerns. This story covers auth token TTL only. Secret KV v2 has no built-in leases (static secrets). Only dynamic secrets (AppRole auth tokens) are in scope here.

- **Renewal semantics:** A token with `token_ttl=3600` and `token_max_ttl=86400` can be renewed up to 8 times (at 1-hour intervals) before it reaches the max TTL cap and must be re-issued via fresh AppRole login. Document this behavior in the TTL policy section so operators understand re-authentication triggers.

### Project Structure Notes

- Modifies `modules/vault-config/main.tf` (or `roles.tf` if separated) — adds/updates `token_ttl` and `token_max_ttl` on all `vault_approle_auth_backend_role` resources
- Creates or updates `docs/runbooks/workload-onboarding.md` — adds `## Token TTL Policy` section

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 3.2]
- [Source: docs/terminus/infra/secrets/architecture.md#AppRole Vault Integration — AR-S8]
- [Source: docs/terminus/infra/secrets/architecture.md#Non-Functional Requirements — NFR3]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
