# Story 2.2: Create Semaphore Ingress and cert-manager Certificate

Status: ready-for-dev

## Story

As a platform operator,
I want HTTPS access to Semaphore UI at `https://semaphore.trantor.internal` with a valid cert-manager-issued TLS certificate,
So that the Semaphore web interface is accessible from the homelab network with proper TLS.

## Acceptance Criteria

1. **Given** the Semaphore Deployment is `Running` (Story 2.1 complete) AND DNS A-record `semaphore.trantor.internal → 10.0.0.126` exists
   **When** the Certificate and Ingress manifests are applied (via ArgoCD or kubectl)
   **Then** `kubectl get certificate semaphoreui-tls-cert -n terminus-infra` shows `READY: True`
2. `curl -sk https://semaphore.trantor.internal | grep -i semaphore` returns content (Semaphore UI HTML)
3. `curl -v https://semaphore.trantor.internal 2>&1 | grep "issuer"` shows the Vault PKI issuer CN
4. HTTP→HTTPS redirect works (HTTP request is redirected to HTTPS)

## Tasks

- [ ] Task 1: Add DNS record (operator prerequisite — must complete before cert issuance)
  - [ ] Synology DNS Manager: add A-record `semaphore.trantor.internal → 10.0.0.126`
  - [ ] Verify: `dig semaphore.trantor.internal @<synology-ip>` returns `10.0.0.126`
  - [ ] Verify: `dig semaphore.trantor.internal` from cluster node also resolves correctly
- [ ] Task 2: Create `certificate.yaml`
  - [ ] File: `platforms/k3s/manifests/terminus-infra/semaphoreui/certificate.yaml`
  - [ ] `apiVersion: cert-manager.io/v1`, `kind: Certificate`
  - [ ] `metadata.name: semaphoreui-tls-cert`, `namespace: terminus-infra`
  - [ ] `spec.issuerRef.kind: ClusterIssuer`, `spec.issuerRef.name: vault-pki`
  - [ ] `spec.dnsNames: [semaphore.trantor.internal]`
  - [ ] `spec.secretName: semaphoreui-tls`
- [ ] Task 3: Create `ingress.yaml`
  - [ ] File: `platforms/k3s/manifests/terminus-infra/semaphoreui/ingress.yaml`
  - [ ] `metadata.name: semaphoreui`, `namespace: terminus-infra`
  - [ ] `spec.ingressClassName: traefik`
  - [ ] `spec.tls[]: hosts: [semaphore.trantor.internal], secretName: semaphoreui-tls`
  - [ ] `spec.rules[]: host: semaphore.trantor.internal` → `backend.service.name: semaphoreui`, `port.number: 3000`
- [ ] Task 4: Verify end-to-end TLS
  - [ ] Wait for cert issuance: `kubectl get certificate semaphoreui-tls-cert -n terminus-infra -w`
  - [ ] If cert stalls: `kubectl describe certificate semaphoreui-tls-cert -n terminus-infra` — check CertificateRequest events
  - [ ] Test: `curl -sk https://semaphore.trantor.internal` returns Semaphore login page content
  - [ ] Test: `curl -v https://semaphore.trantor.internal 2>&1 | grep -i issuer` — confirm vault-pki CA

## Dev Notes

### DNS Prerequisite

DNS is a **hard prerequisite** for cert-manager Certificate issuance. The `vault-pki` ClusterIssuer uses an internal DNS-based validation path. Without the DNS record, the CertificateRequest will fail or stall. Complete Task 1 before applying Certificate/Ingress manifests.

### ClusterIssuer Pre-Check

Before applying, confirm `vault-pki` is healthy:
```bash
kubectl get clusterissuer vault-pki
```
Should show `READY: True`. If it's not ready, the Certificate will fail immediately.

### HTTP→HTTPS Redirect

HTTP→HTTPS redirect is handled by Traefik platform defaults (IngressClass `traefik` has redirect middleware enabled globally). No per-Ingress annotation is needed. Do NOT add manual redirect annotations unless the platform default is disabled.

### TLS Termination

Traefik terminates TLS at the edge. The backend service `semaphoreui` receives plain HTTP traffic on port 3000. The `spec.tls` section in the Ingress configures Traefik to present the cert-manager-issued certificate for `semaphore.trantor.internal`.

### Certificate Issuance Timing

cert-manager + vault-pki typically issues within 30–60 seconds in the homelab. If it exceeds 2 minutes:
1. `kubectl describe certificaterequest -n terminus-infra` — look for error events
2. Check Vault connectivity: `kubectl logs -n cert-manager deployment/cert-manager | tail -20`
3. Verify `vault-pki` ClusterIssuer token is still valid (may need Vault token renewal)

### Architecture Reference

- [Source: docs/terminus/infra/semaphoreui/architecture.md — ArgoCD Application and Ingress sections]
- [Source: docs/terminus/infra/semaphoreui/tech-decisions.md — TD-005 (TLS via cert-manager), TD-006 (Traefik IngressClass)]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (GitHub Copilot)

### Debug Log References

### Completion Notes List

### File List

- `platforms/k3s/manifests/terminus-infra/semaphoreui/certificate.yaml` (new, in `terminus.infra`)
- `platforms/k3s/manifests/terminus-infra/semaphoreui/ingress.yaml` (new, in `terminus.infra`)
