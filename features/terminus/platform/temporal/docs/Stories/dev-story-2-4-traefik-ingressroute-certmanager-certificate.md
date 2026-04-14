# Story 2.4: Traefik IngressRoute + cert-manager Certificate for Temporal Web UI

Status: done

## Story

As a platform developer,
I want the Temporal Web UI exposed via Traefik on `temporal-ui.trantor.internal` with TLS issued by cert-manager Vault PKI,
so that the Web UI is accessible within the network using the same ingress and TLS pattern as all other terminus k3s services.

## Acceptance Criteria

1. `k8s/ingress-route.yaml` created (or Helm IngressRoute template matching existing terminus pattern):
   - `kind: IngressRoute` (Traefik CRD)
   - `spec.routes[0].match: "Host(\`temporal-ui.trantor.internal\`)"`
   - `spec.routes[0].services[0].name: temporal-web` (Temporal chart service name)
   - `spec.routes[0].services[0].port: 8080`
   - `spec.tls.secretName: temporal-ui-tls`
2. `k8s/certificate.yaml` created:
   - `kind: Certificate`
   - `spec.dnsNames: [temporal-ui.trantor.internal]`
   - `spec.issuerRef.name: {vault-pki-issuer}` (follows existing terminus cert pattern)
   - `spec.issuerRef.kind: ClusterIssuer`
   - `spec.secretName: temporal-ui-tls`
3. Both resources in `temporal` namespace
4. `https://temporal-ui.trantor.internal` accessible in browser, responds with Temporal Web UI
5. Certificate is valid and trusted (issued by Vault PKI)
6. Vault PKI ClusterIssuer name documented (follows pattern from existing k3s services)

## Tasks / Subtasks

- [ ] Task 1: Resolve Vault PKI ClusterIssuer name (AC: 6)
  - [ ] Check existing terminus k3s services for cert-manager Certificate resources
  - [ ] `kubectl get clusterissuer -A` to list all ClusterIssuers
  - [ ] Identify the Vault PKI ClusterIssuer name used by other terminus services
  - [ ] Document the resolved name (e.g., `vault-issuer`, `terminus-pki`) in a comment in `certificate.yaml`

- [ ] Task 2: Create `k8s/certificate.yaml` (AC: 2, 3, 5, 6)
  - [ ] Create Certificate manifest:
    - `apiVersion: cert-manager.io/v1`
    - `kind: Certificate`
    - `namespace: temporal`
    - `spec.dnsNames: [temporal-ui.trantor.internal]`
    - `spec.issuerRef.name: {resolved-clusterissuer}`, `kind: ClusterIssuer`
    - `spec.secretName: temporal-ui-tls`
  - [ ] Add comment at top: `# ClusterIssuer: {name} (resolved from existing terminus cert pattern)`
  - [ ] Commit: `feat(k8s): add cert-manager Certificate for temporal-ui.trantor.internal`

- [ ] Task 3: Create `k8s/ingress-route.yaml` (AC: 1, 3)
  - [ ] Create IngressRoute manifest:
    - `apiVersion: traefik.io/v1alpha1`
    - `kind: IngressRoute`
    - `namespace: temporal`
    - `spec.entryPoints: [websecure]`
    - `spec.routes[0].kind: Rule`
    - `spec.routes[0].match: "Host(\`temporal-ui.trantor.internal\`)"`
    - `spec.routes[0].services[0].name: temporal-web`, `port: 8080`
    - `spec.tls.secretName: temporal-ui-tls`
  - [ ] Commit: `feat(k8s): add Traefik IngressRoute for Temporal Web UI`

- [ ] Task 4: Verify UI accessibility (AC: 4, 5)
  - [ ] Apply Certificate: `kubectl apply -f k8s/certificate.yaml`
  - [ ] Verify Certificate issued: `kubectl describe certificate temporal-ui-tls -n temporal`
  - [ ] Apply IngressRoute: `kubectl apply -f k8s/ingress-route.yaml`
  - [ ] Open `https://temporal-ui.trantor.internal` in browser
  - [ ] Confirm Temporal Web UI loads and certificate is valid

## Dev Notes

### Target Repo Path
```
TargetProjects/terminus/platform/terminus.platform/
```

### Architecture Reference
Source: Architecture Step 4 (ingress hostname `temporal-ui.trantor.internal`, TLS via cert-manager Vault PKI)

### Traefik API
IngressRoute is a Traefik CRD (`traefik.io/v1alpha1`), not a standard k8s `Ingress` resource. Use `spec.tls.secretName` to reference the cert-manager-issued Secret.

### Story Scope
Only IngressRoute and Certificate. No auth middleware, rate limiting, or other Traefik middleware in this story.

### ClusterIssuer Name
Must follow the established terminus pattern. Reference: check how other k3s services (semaphoreui, etc.) declare their cert-manager Certificates and use the same ClusterIssuer.

### Constitutional Override — Develop-First Integration
Implementation work lands on `develop` in `terminus.platform` (Terminus Art. 5 override active).

### Not In Scope
- Auth or rate limiting middleware
- Prometheus metrics for the Web UI
- Any modification to Temporal server configuration

## Pre-Implementation Constitution Gates

| Gate | Article | Result |
|------|---------|--------|
| Track declared | Org Art. 1 | ✅ PASS — `tech-change` |
| Architecture documented | Org Art. 3 | ✅ PASS — `docs/terminus/platform/temporal/architecture.md` |
| No confidential data | Org Art. 4 | ✅ PASS — no credentials, TLS managed by cert-manager |
| Git discipline | Org Art. 5 | ✅ PASS — develop-first integration authorized by terminus Art. 5 |
| Security first | Org Art. 9 | ✅ PASS — TLS required, Vault PKI issuer |
| Repo as source of truth | Org Art. 10 | ✅ PASS — all context from committed docs |
| Develop-first integration | Terminus Art. 5 | ✅ OVERRIDE ACTIVE — land implementation work on `develop` in `terminus.platform` |

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (copilot)

### Debug Log References

### Completion Notes List

### File List
