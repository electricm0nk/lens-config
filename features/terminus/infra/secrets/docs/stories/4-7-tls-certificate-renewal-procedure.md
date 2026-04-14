# Story 4.7: TLS Certificate Renewal Procedure — Operational Runbook (NFR7, FR36)

Status: ready-for-dev

## Story

As the operator,
I want a documented TLS certificate renewal procedure that walks through checking expiry, renewing via `tofu apply`, and verifying the renewed certificate,
so that TLS certificate maintenance is non-disruptive and executable by any operator following the runbook.

## Acceptance Criteria

1. **[Runbook exists at canonical path]** Given `docs/runbooks/tls-renewal.md` is created, when I read it, then it contains the following sections: `## Checking Certificate Expiry`, `## Renewal Procedure`, `## Post-Renewal Verification`, `## Consumer Re-Trust`, and `## Renewal Schedule`.

2. **[Expiry check command documented and executable]** Given the runbook `## Checking Certificate Expiry` section, when I run the documented command against the live Vault TLS certificate, then the output shows the certificate's expiry date in a human-readable format. The command should be:
   ```sh
   openssl x509 -enddate -noout -in /etc/vault.d/tls/ca.crt
   ```
   or equivalent (e.g., using `openssl s_client` to check live endpoint).

3. **[Renewal procedure is tofu-driven]** Given the `## Renewal Procedure` section, when I follow it, then the documented renewal steps include:
   - `tofu plan -target module.infra.tls_private_key.ca -target module.infra.tls_self_signed_cert.ca` — preview scope
   - `tofu apply -target module.infra` — apply TLS resources only
   - The runbook notes that targeting `module.infra` regenerates the CA key and cert and all dependent server certs

4. **[tofu plan shows only TLS cert changes]** Given the renewal procedure is followed, when I run `tofu plan -target module.infra`, then the plan shows changes ONLY to TLS resources (tls_private_key, tls_self_signed_cert, tls_locally_signed_cert) — no unintended VM reprovisioning or Vault backend changes.

5. **[Post-renewal verification passes]** Given the TLS certificate is renewed and the new CA cert is distributed to the operator machine, when I run:
   ```sh
   curl --cacert <new-ca.pem> https://<vault_addr>/v1/sys/health
   ```
   then the response is HTTP 200 (Vault is healthy and TLS is valid with the new cert).

6. **[Consumer re-trust requirement documented]** Given the `## Consumer Re-Trust` section, when I read it, then it documents that after CA cert renewal, all consumers (Ansible inventory `vault_ca_cert_path`, OpenTofu provider config, shell `VAULT_CACERT`) must receive the new CA cert and be restarted or reconfigured before they can connect to Vault.

7. **[Renewal schedule documented]** Given the `## Renewal Schedule` section, when I read it, then it states: "Self-signed CA cert validity is set to <N> days (per Story 1.3). Renew at 80% of validity period. Record each renewal in `docs/ops-log.md` with the old and new expiry dates."

## Tasks / Subtasks

- [ ] **Task 1: Determine TLS cert validity period from Story 1.3** (AC: 7)
  - [ ] Read `modules/vault-infra/main.tf` (or wherever the tls_self_signed_cert resource is) to find the `validity_period_hours` value
  - [ ] Convert to days: `validity_period_hours / 24` → record as `<N>` days
  - [ ] Record the certificate creation date from Story 1.3 deployment to calculate current expiry

- [ ] **Task 2: Create `docs/runbooks/tls-renewal.md`** (AC: 1, 2, 3, 6, 7)
  - [ ] Create the runbook with full content:
    ```markdown
    # TLS Certificate Renewal Runbook

    ## Overview
    The Vault TLS infrastructure uses a self-signed CA (D6 architecture decision).
    Certificates are managed by OpenTofu's `hashicorp/tls` provider in the vault-infra module.
    Renewal replaces both the CA private key and all issued certificates.

    ## Checking Certificate Expiry

    Check the CA certificate expiry on the Vault VM:
    ```sh
    openssl x509 -enddate -noout -in /etc/vault.d/tls/ca.crt
    # Example output: notAfter=Mar 24 12:00:00 2027 GMT
    ```

    Check the live cert on the Vault endpoint:
    ```sh
    echo | openssl s_client -connect <vault_ip>:8200 -servername <vault_ip> 2>/dev/null \
      | openssl x509 -noout -enddate
    ```

    Check from the OpenTofu state:
    ```sh
    tofu state show 'module.infra.tls_self_signed_cert.ca' | grep validity
    ```

    Renewal trigger: Renew when less than 20% of validity period remains.
    For a <N>-day cert: renew at day <0.8*N>.

    ## Renewal Procedure

    ### Step 1: Review the plan
    ```sh
    tofu plan -target module.infra
    ```
    Confirm plan shows only TLS resource changes (tls_private_key, tls_self_signed_cert, tls_locally_signed_cert).
    Expected: no VM reprovisioning, no Vault backend changes, no KV secret changes.

    ### Step 2: Taint TLS resources (force regeneration)
    OpenTofu tls resources regenerate on `apply` if their validity period has changed or if tainted.
    To force regeneration without changing validity_period_hours:
    ```sh
    tofu taint 'module.infra.tls_private_key.ca'
    tofu taint 'module.infra.tls_self_signed_cert.ca'
    ```

    ### Step 3: Apply
    ```sh
    tofu apply -target module.infra
    ```
    This regenerates the CA key, CA cert, and all server certs signed by the CA.

    ### Step 4: Retrieve the new CA certificate
    ```sh
    tofu output -raw vault_ca_cert > ca-new.pem
    ```

    ## Post-Renewal Verification

    ### Verify Vault health with new cert
    ```sh
    curl --cacert ca-new.pem https://<vault_ip>:8200/v1/sys/health
    # Expected: {"initialized":true,"sealed":false,...}
    ```

    ### Verify via shell with VAULT_CACERT
    ```sh
    export VAULT_CACERT=ca-new.pem
    vault status
    # Expected: Initialized: true, Sealed: false
    ```

    ## Consumer Re-Trust

    After CA cert renewal, ALL consumers must receive and trust the new CA cert:

    | Consumer | Update Action |
    |---|---|
    | Operator shell | Update `VAULT_CACERT` env var to point to `ca-new.pem` |
    | Ansible inventory | Update `vault_ca_cert_path` on Vault VM: `scp ca-new.pem vault_vm:/etc/vault.d/tls/ca.crt` |
    | OpenTofu provider | `tofu apply` on vault-config module after cert renewal (uses updated provider) |
    | k3s workload | Re-distribute CA cert to k3s cluster trust store |
    | openclaw agent | Update CA cert in agent configuration |

    **WARNING:** Until consumers update their CA trust, they will receive TLS verification errors.
    Ensure all consumers are updated before the old cert expires.

    ## Renewal Schedule

    Self-signed CA cert validity: <N> days (set in vault-infra module, tls_self_signed_cert.ca).
    Renewal target: Day <0.8 * N> after issuance.

    Record each renewal in `docs/ops-log.md`:
    ```markdown
    ## TLS Certificate Renewal — <Date>
    - Old expiry: <old_expiry>
    - New expiry: <new_expiry>
    - Operator: <name>
    - Consumer re-trust: all consumers updated ✅
    ```

    ## Reference
    - Architecture decision D6: self-signed CA (docs/terminus/infra/secrets/architecture.md)
    - Story 1.3: initial TLS cert generation
    - Story 1.8: root module outputs including vault_ca_cert
    ```

- [ ] **Task 3: Validate tofu plan scope for TLS renewal** (AC: 4)
  - [ ] Run: `tofu plan -target module.infra` (or equivalent target for tls resources)
  - [ ] Inspect plan output — confirm only TLS resources are in the diff (no VM, no Vault backend)
  - [ ] Document the exact resource addresses under the `## Renewal Procedure` section if they differ from the template above

- [ ] **Task 4: Test post-renewal verification command** (AC: 5)
  - [ ] After obtaining the current CA cert: `tofu output -raw vault_ca_cert > ca-current.pem`
  - [ ] Run: `curl --cacert ca-current.pem https://<vault_ip>:8200/v1/sys/health` → HTTP 200
  - [ ] Confirm the verification command works in the current cert state (demonstrates the same command will work after renewal)

## Dev Notes

### Architecture Constraints

- **D6 — Self-signed TLS CA:** The TLS infrastructure uses a self-signed CA generated by `hashicorp/tls` in the vault-infra module (Story 1.3). There is no ACME/Let's Encrypt renewal process. Renewal is a full regeneration of the CA key + issuing new server certs. This is simpler than ACME but requires manual coordination.
  [Source: docs/terminus/infra/secrets/architecture.md#Infrastructure Decisions — D6]

- **`tofu taint` approach:** The `tls_private_key` and `tls_self_signed_cert` resources regenerate when tainted or when their configuration changes (e.g., changing `validity_period_hours`). For a clean renewal without changing config, use `tofu taint` to force regeneration. Document this explicitly in the runbook.

- **Consumer re-trust is critical:** The biggest operational risk in TLS cert renewal is forgetting to update consumers. If Ansible, OpenTofu provider, or k3s is configured with the OLD CA cert, they will fail TLS verification immediately after renewal. The Consumer Re-Trust table in the runbook must list all known consumers.

- **NFR7 — TLS on all Vault API endpoints:** NFR7 requires TLS on all Vault API communication. The renewal procedure maintains this by generating a new valid cert before the old one expires. Do not let the cert expire — this would take down all consumers.

### Project Structure Notes

- Creates `docs/runbooks/tls-renewal.md` (new file)
- No changes to infrastructure code

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 4.7]
- [Source: docs/terminus/infra/secrets/architecture.md#Infrastructure Decisions — D6]
- [Source: docs/terminus/infra/secrets/architecture.md#Non-Functional Requirements — NFR7]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
