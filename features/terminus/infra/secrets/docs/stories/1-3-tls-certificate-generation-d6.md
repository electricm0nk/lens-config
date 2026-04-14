# Story 1.3: TLS Certificate Generation (D6)

Status: ready-for-dev

## Story

As the operator,
I want the `infra/` module to generate a self-signed CA and server certificate for Vault's TLS listener,
so that all Vault API traffic is encrypted from the first `tofu apply` with no manual cert management.

## Acceptance Criteria

1. **[CA and server cert generated]** Given `modules/infra/main.tf` with the `hashicorp/tls` provider, when I run `tofu apply -target module.infra`, then a self-signed CA private key and certificate are generated, and a Vault server certificate signed by that CA is generated for the VM's static IP address.

2. **[Outputs declared correctly]** Given `modules/infra/outputs.tf`, when I inspect it, then the following outputs are declared:
   - `ca_cert_pem` — NOT marked sensitive (public trust anchor; committed to git)
   - `ca_private_key_pem` — marked `sensitive = true`
   - `server_cert_pem` — NOT marked sensitive (public, distributed to consumers)
   - `server_private_key_pem` — marked `sensitive = true`

3. **[CA cert consumer distribution]** Given the `ca_cert_pem` output description field in `modules/infra/outputs.tf`, when I read it, then it states: "CA public certificate — committed to git unencrypted for consumer trust distribution (Ansible, OpenTofu vault provider, k3s)".

4. **[Server cert covers VM IP]** Given the generated server certificate, when I inspect the cert Subject Alternative Names, then the static `vm_ip` is listed as an IP SAN so that TLS verification succeeds against the Vault listener address `https://{vm_ip}:8200`.

5. **[No cert re-generation on plan]** Given a clean `tofu apply` of the infra module, when I run `tofu plan -target module.infra` a second time, then the plan shows no changes to the TLS resources (idempotent — NFR16).

6. **[CA cert committed to git]** Given the `docs/runbooks/day0-bootstrap.md` (Story 1.8), when I read the post-apply steps, then there is a step instructing the operator to copy `tofu output vault_ca_cert_pem` to a committed file (e.g., `ca.crt` at repo root) so consumers can reference it.

## Tasks / Subtasks

- [ ] **Task 1: Add TLS resources to `modules/infra/main.tf`** (AC: 1, 4)
  - [ ] Add `tls_private_key.ca` resource (algorithm: RSA, rsa_bits: 4096)
  - [ ] Add `tls_self_signed_cert.ca` resource referencing `tls_private_key.ca`; set `is_ca_certificate = true`; set validity period (e.g., `validity_period_hours = 87600` for 10 years — homelab CA, not rotated frequently); set `allowed_uses = ["cert_signing", "key_encipherment", "digital_signature"]`
  - [ ] Add `tls_private_key.server` resource (RSA 4096)
  - [ ] Add `tls_cert_request.server` resource using `tls_private_key.server`; include IP SAN from `var.vm_ip`
  - [ ] Add `tls_locally_signed_cert.server` resource signed by CA key+cert; set `validity_period_hours = 8760` (1 year); set `allowed_uses = ["server_auth", "key_encipherment", "digital_signature"]`

- [ ] **Task 2: Update `modules/infra/outputs.tf`** (AC: 2, 3)
  - [ ] Add `ca_cert_pem` output (`tls_self_signed_cert.ca.cert_pem`) — NOT sensitive; description: "CA public certificate — committed to git unencrypted for consumer trust distribution (Ansible, OpenTofu vault provider, k3s)"
  - [ ] Add `ca_private_key_pem` output — `sensitive = true`; description: "CA private key — written to SOPS-encrypted state only; never committed plaintext"
  - [ ] Add `server_cert_pem` output — NOT sensitive; description: "Vault server TLS certificate signed by CA"
  - [ ] Add `server_private_key_pem` output — `sensitive = true`; description: "Vault server TLS private key"
  - [ ] Keep existing `vm_ip` output from Story 1.2

- [ ] **Task 3: Update root `outputs.tf`** (AC: 6)
  - [ ] Wire `vault_ca_cert_pem = module.infra.ca_cert_pem` (replaces TODO stub from Story 1.1)
  - [ ] Ensure `vault_ca_cert_pem` is NOT sensitive at root level (public trust anchor)

- [ ] **Task 4: Verify IP SAN coverage** (AC: 4)
  - [ ] After apply, pipe `tofu output -raw vault_ca_cert_pem` to `openssl x509 -noout -text | grep -A2 "Subject Alternative"`
  - [ ] Confirm the VM static IP appears as an IP SAN entry
  - [ ] If DNS name is also needed (future), add `dns_names` to the `tls_cert_request.server` resource now (leave as empty list if not needed)

- [ ] **Task 5: Confirm idempotency** (AC: 5)
  - [ ] Run `tofu plan -target module.infra` after clean apply
  - [ ] Confirm no changes to `tls_private_key.*`, `tls_self_signed_cert.*`, `tls_locally_signed_cert.*`

## Dev Notes

### Architecture Constraints

- **D6 — TLS strategy (architecture decision):** Self-signed CA and server cert generated entirely via OpenTofu `tls` provider. CA private key is protected via SOPS-encrypted state (D4). CA public cert is committed to git unencrypted for consumer trust distribution. This is the authoritative source-of-truth for D6.
  [Source: docs/terminus/infra/secrets/architecture.md#TLS & Certificates — D6]

- **CA cert sensitivity rules:** `ca_cert_pem` is intentionally NOT marked sensitive — it is a public trust anchor. Marking it sensitive would prevent its use in downstream module `depends_on` expressions and complicate provider TLS configuration. `ca_private_key_pem` IS sensitive and goes into SOPS-encrypted state (D4).
  [Source: docs/terminus/infra/secrets/architecture.md#TLS & Certificates — D6]

- **IP SAN requirement:** Vault's TLS listener binds to the VM's static IP. The server cert must include the IP as a SAN (`ip_addresses = [var.vm_ip]` in `tls_cert_request.server`). Without this, Vault clients doing TLS verification would fail certificate validation.
  [Source: docs/terminus/infra/secrets/architecture.md#Core Architectural Decisions — D6, Implementation sequence step 1]

- **Provider note:** `hashicorp/tls` provider resources are part of the `infra/` module, consistent with provider isolation rules (no TLS provider calls in `vault-core/` or `vault-config/`). The root `main.tf` already declares the `tls` provider; `module "infra"` inherits it.
  [Source: docs/terminus/infra/secrets/architecture.md#Structure Patterns — OpenTofu Module Layout]

- **Cert renewal:** TLS cert renewal is covered in Story 4.7. The architecture note is: "When `terminus-infra-monitoring` introduces Step-CA or external PKI, Vault provider config is updated to reference new certs — no rearchitecting required." For now, self-signed with a long validity period is correct.
  [Source: docs/terminus/infra/secrets/architecture.md#TLS & Certificates — D6 Upgrade path]

### Project Structure Notes

- This story modifies existing files in `modules/infra/` (already created as stubs in Story 1.1, populated in Story 1.2). Do NOT create new files — only modify `modules/infra/main.tf`, `modules/infra/outputs.tf`, and root `outputs.tf`.
- The `ca.crt` file committed to git (as referenced in AC 6) is a Day-0 operator step, not an automated file creation — it is documented in the runbook (Story 1.8), not created by this story.

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 1.3]
- [Source: docs/terminus/infra/secrets/architecture.md#TLS & Certificates — D6]
- [Source: docs/terminus/infra/secrets/architecture.md#Complete Project Directory Structure]
- [Source: docs/terminus/infra/secrets/architecture.md#Format Patterns — OpenTofu Outputs Public Contract]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
