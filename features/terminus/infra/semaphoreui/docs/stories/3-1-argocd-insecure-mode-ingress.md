# Story 3.1: Configure ArgoCD Insecure Mode and Deploy ArgoCD Ingress

Status: ready-for-dev

## Story

As a platform operator,
I want the ArgoCD UI accessible at `https://argocd.trantor.internal` with TLS,
So that ArgoCD can be used from the homelab network browser without port-forwarding.

## Acceptance Criteria

1. **Given** ArgoCD is operational (ClusterIP only) AND DNS A-record `argocd.trantor.internal → 10.0.0.126` exists AND `vault-pki` ClusterIssuer is healthy
   **When** the `argocd-cmd-params-cm` patch is applied, argocd-server is restarted, and Certificate + Ingress manifests are applied via kubectl
   **Then** `kubectl get certificate argocd-tls-cert -n argocd` shows `READY: True`
2. `curl -sk https://argocd.trantor.internal | grep -i "argo"` returns ArgoCD UI HTML content
3. The ArgoCD web interface loads the login page at `https://argocd.trantor.internal` without certificate warnings in a browser trusting the Vault PKI CA
4. ArgoCD continues to sync all existing Applications normally after the insecure mode change (no regression)

## Tasks

- [ ] Task 1: Add DNS record (operator prerequisite)
  - [ ] Synology DNS Manager: add A-record `argocd.trantor.internal → 10.0.0.126`
  - [ ] Verify: `dig argocd.trantor.internal` returns `10.0.0.126`
- [ ] Task 2: Patch `argocd-cmd-params-cm` ConfigMap to enable insecure mode
  - [ ] Create `platforms/k3s/manifests/argocd/configmap-argocd-cmd-params.yaml` in `terminus.infra`
    ```yaml
    # NOTE: This file is manually applied via kubectl (NOT managed by ArgoCD).
    # Do NOT add this file to any ArgoCD Application or App-of-Apps.
    # Reason: ArgoCD cannot manage its own infrastructure components that affect its own accessibility.
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: argocd-cmd-params-cm
      namespace: argocd
    data:
      server.insecure: "true"
    ```
  - [ ] Apply: `kubectl apply -f platforms/k3s/manifests/argocd/configmap-argocd-cmd-params.yaml`
  - [ ] Verify: `kubectl get cm argocd-cmd-params-cm -n argocd -o jsonpath='{.data.server\.insecure}'` returns `true`
- [ ] Task 3: Restart ArgoCD server to activate insecure mode
  - [ ] `kubectl rollout restart deployment argocd-server -n argocd`
  - [ ] Wait for rollout: `kubectl rollout status deployment argocd-server -n argocd` — confirm `Rolled out`
  - [ ] Verify insecure mode active: ArgoCD server logs should no longer show TLS listener (check with `kubectl logs -n argocd deployment/argocd-server | grep -i tls | head -5`)
- [ ] Task 4: Create `platforms/k3s/manifests/argocd/certificate.yaml` in `terminus.infra`
  - [ ] `apiVersion: cert-manager.io/v1`, `kind: Certificate`
  - [ ] `metadata.name: argocd-tls-cert`, `namespace: argocd`
  - [ ] `spec.issuerRef.kind: ClusterIssuer`, `spec.issuerRef.name: vault-pki`
  - [ ] `spec.dnsNames: [argocd.trantor.internal]`
  - [ ] `spec.secretName: argocd-tls`
  - [ ] Add comment in the file: `# Manually applied (not managed by ArgoCD — bootstrapping constraint)`
- [ ] Task 5: Create `platforms/k3s/manifests/argocd/ingress.yaml` in `terminus.infra`
  - [ ] `metadata.name: argocd`, `namespace: argocd`, `spec.ingressClassName: traefik`
  - [ ] `spec.tls[]: hosts: [argocd.trantor.internal], secretName: argocd-tls`
  - [ ] `spec.rules[]: host: argocd.trantor.internal` → `backend.service.name: argocd-server`, `port.number: 80` (plain HTTP — insecure mode)
  - [ ] Add comment in the file: `# Manually applied (not managed by ArgoCD — bootstrapping constraint)`
- [ ] Task 6: Apply ArgoCD manifests via kubectl (NOT via ArgoCD App-of-Apps)
  - [ ] `kubectl apply -f platforms/k3s/manifests/argocd/certificate.yaml`
  - [ ] `kubectl apply -f platforms/k3s/manifests/argocd/ingress.yaml`
  - [ ] Monitor cert issuance: `kubectl get certificate argocd-tls-cert -n argocd -w` — wait for `READY: True`
- [ ] Task 7: Validate ArgoCD UI access
  - [ ] `curl -sk https://argocd.trantor.internal | grep -i "argo"` — returns ArgoCD HTML
  - [ ] Open `https://argocd.trantor.internal` in a browser — confirm login page loads with no cert warning
  - [ ] Log in to ArgoCD UI — confirm all Applications are visible and healthy
  - [ ] Trigger an ArgoCD sync on one Application — confirm sync works normally (no regression)

## Dev Notes

### Why kubectl and NOT ArgoCD

This story uses `kubectl apply` (NOT ArgoCD) for ALL manifests. This is the **bootstrapping constraint**: ArgoCD cannot manage the Ingress that makes it accessible, because if ArgoCD breaks while trying to apply its own Ingress, the cluster operator loses ArgoCD access entirely. The manifests are committed to `terminus.infra` for gitops auditability but applied outside of ArgoCD's control plane.

**CRITICAL**: Do NOT add `platforms/k3s/manifests/argocd/` to any ArgoCD Application or the App-of-Apps. Future operators must NOT add these files to ArgoCD management. The comments in the manifest files reinforce this.

### Ingress Backend Port Must Be 80 (Not 443)

After enabling insecure mode, the `argocd-server` pod listens on port `80` (plain HTTP) internally. The Ingress backend **must** point to port `80`. If you use port `443` or `8080`, you will get a redirect loop:
- Traefik → (HTTPS) → ArgoCD on 443 → ArgoCD redirects HTTP → Traefik → (HTTPS) → loop

Correct:
```yaml
backend:
  service:
    name: argocd-server
    port:
      number: 80
```

### argocd-cmd-params-cm Merge Behavior

`kubectl apply` on the ConfigMap merges keys with the existing ConfigMap. If `argocd-cmd-params-cm` already exists with other keys, applying the new file preserves those keys and adds `server.insecure: "true"`. This is safe.

### Restart Timing

After the ConfigMap patch, the ArgoCD server must be restarted to pick up the new setting (ConfigMap changes are not live-reloaded). The rollout restart creates a new pod. All existing ArgoCD Application sync operations are briefly interrupted during rollout (typically 10–30 seconds). No synced state is lost.

### Independent of Epic 2

This story is independent of all Epic 2 stories. It can be worked before, during, or after deploying the Semaphore application. However, the ArgoCD UI being accessible makes it much easier to monitor Epic 2 Application sync status — recommended to complete this story early.

### Architecture Reference

- [Source: docs/terminus/infra/semaphoreui/architecture.md — ArgoCD Ingress + Insecure Mode section]
- [Source: docs/terminus/infra/semaphoreui/tech-decisions.md — TD-018 (insecure mode via ConfigMap), TD-019 (ArgoCD Ingress)]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (GitHub Copilot)

### Debug Log References

### Completion Notes List

### File List

- `platforms/k3s/manifests/argocd/configmap-argocd-cmd-params.yaml` (new, in `terminus.infra`)
- `platforms/k3s/manifests/argocd/certificate.yaml` (new, in `terminus.infra`)
- `platforms/k3s/manifests/argocd/ingress.yaml` (new, in `terminus.infra`)
