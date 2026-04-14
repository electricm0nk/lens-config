# Story 1.5: Helm Chart and ArgoCD Deployment to k3s

Status: ready-for-dev

## Story

As Todd,
I want the API deployed to Terminus k3s via ArgoCD with ESO-managed secrets,
so that the platform infrastructure is live, GitOps-driven, and ready for business logic.

**Prerequisite:** `terminus-infra-secrets` and `terminus-infra-k3s` must be live on the target cluster.

## Acceptance Criteria

1. `deploy/helm/fourdogs-central/templates/deployment.yaml` runs `cmd/migrate` as a k8s init container (must exit 0 before API pod is Ready)
2. `templates/external-secret-ghcr.yaml` (sync-wave `-1`) provisions `ghcr-pull-secret` from `secret/terminus/default/ghcr/pull-token`
3. `templates/externalsecret.yaml` (sync-wave `0`) provisions k8s Secret from ESO/Vault: `DATABASE_URL`, `OAUTH_CLIENT_ID`, `OAUTH_CLIENT_SECRET`, `CORS_ALLOWED_ORIGIN` — Vault path: `secret/terminus/fourdogs/central`
4. `deployment.yaml` includes `imagePullSecrets: [{name: ghcr-pull-secret}]` — required for GHCR image pull
5. `templates/ingressroute.yaml` configures Traefik IngressRoute with cert-manager TLS certificate
6. `templates/service.yaml` exposes the API pod on correct internal port
7. `GET /v1/health` returns `HTTP 200 {"status": "ok"}` on live cluster after sync
8. ArgoCD shows Application as `Synced` and `Healthy`
9. Vault secrets seeded via `seed-fourdogs-central-secrets.yml` before first ArgoCD sync

## Tasks / Subtasks

- [ ] Task 0: Verify Story 1.4 complete — Docker image pushed to GHCR successfully

- [ ] Task 1: Create Vault seed playbook pre-condition (AC: 9)
  - [ ] Create `terminus.infra/platforms/k3s/ansible/playbooks/seed-fourdogs-central-secrets.yml`
    - Seeds `secret/terminus/fourdogs/central` with: `DATABASE_URL`, `OAUTH_CLIENT_ID`, `OAUTH_CLIENT_SECRET`, `CORS_ALLOWED_ORIGIN`
  - [ ] Run via Semaphore against `automation-dev` BEFORE triggering ArgoCD sync
  - [ ] **Do not proceed to Helm/ArgoCD steps until this playbook has been run successfully**

- [ ] Task 1b: Initialize Helm chart structure (AC: all)
  - [ ] Create `deploy/helm/fourdogs-central/Chart.yaml`:
    ```yaml
    apiVersion: v2
    name: fourdogs-central
    description: fourdogs-central catalog API
    type: application
    version: 0.1.0
    appVersion: "latest"
    ```
  - [ ] Create `deploy/helm/fourdogs-central/values.yaml`:
    ```yaml
    image:
      repository: ghcr.io/electricm0nk/fourdogs-central
      tag: ""  # overridden by ArgoCD image updater or manual sync
      pullPolicy: IfNotPresent

    service:
      port: 8080

    ingress:
      host: fourdogs.terminus.local  # update to actual hostname

    vault:
      path: secret/terminus/fourdogs/central  # Vault path per architecture hierarchy
    ```
  - [ ] Commit: `feat(helm): initialize helm chart structure`

- [ ] Task 2: Create `templates/deployment.yaml` with migrate init container (AC: 1, 4)
  - [ ] Create `deploy/helm/fourdogs-central/templates/deployment.yaml`:
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: {{ include "fourdogs-central.fullname" . }}
      labels:
        {{- include "fourdogs-central.labels" . | nindent 4 }}
    spec:
      replicas: 1
      selector:
        matchLabels:
          {{- include "fourdogs-central.selectorLabels" . | nindent 6 }}
      template:
        metadata:
          labels:
            {{- include "fourdogs-central.selectorLabels" . | nindent 8 }}
        spec:
          imagePullSecrets:
            - name: ghcr-pull-secret
          initContainers:
            - name: migrate
              image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
              command: ["/migrate", "up"]
              envFrom:
                - secretRef:
                    name: fourdogs-central-secrets
          containers:
            - name: api
              image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
              ports:
                - containerPort: {{ .Values.service.port }}
              envFrom:
                - secretRef:
                    name: fourdogs-central-secrets
              readinessProbe:
                httpGet:
                  path: /v1/health
                  port: {{ .Values.service.port }}
                initialDelaySeconds: 5
                periodSeconds: 10
              livenessProbe:
                httpGet:
                  path: /v1/health
                  port: {{ .Values.service.port }}
                initialDelaySeconds: 15
                periodSeconds: 20
    ```
  - [ ] Create `deploy/helm/fourdogs-central/templates/_helpers.tpl` with standard fullname/labels helpers
  - [ ] Commit: `feat(helm): add deployment with migrate init container`

- [ ] Task 2b: Create GHCR pull secret ExternalSecret (AC: 2) — sync-wave -1
  - [ ] Create `deploy/helm/fourdogs-central/templates/external-secret-ghcr.yaml`:
    ```yaml
    apiVersion: external-secrets.io/v1beta1
    kind: ExternalSecret
    metadata:
      name: ghcr-pull-secret
      annotations:
        argocd.argoproj.io/sync-wave: "-1"
    spec:
      refreshInterval: 1h
      secretStoreRef:
        kind: ClusterSecretStore
        name: vault-backend
      target:
        name: ghcr-pull-secret
        template:
          type: kubernetes.io/dockerconfigjson
          data:
            .dockerconfigjson: >
              {"auths":{"ghcr.io":{"username":"electricm0nk","password":"{{ .token }}"}}}
      data:
        - secretKey: token
          remoteRef:
            key: secret/terminus/default/ghcr/pull-token
            property: token
    ```
  > The Vault path `secret/terminus/default/ghcr/pull-token` is **shared** — already seeded, no re-seeding required.
  - [ ] Commit: `feat(helm): add GHCR pull secret ExternalSecret (wave -1)`

- [ ] Task 3: Create `templates/externalsecret.yaml` (AC: 3) — sync-wave 0
  - [ ] Create `deploy/helm/fourdogs-central/templates/externalsecret.yaml`:
    ```yaml
    apiVersion: external-secrets.io/v1beta1
    kind: ExternalSecret
    metadata:
      name: fourdogs-central-secrets
      annotations:
        argocd.argoproj.io/sync-wave: "0"
    spec:
      refreshInterval: 1h
      secretStoreRef:
        name: vault-backend
        kind: ClusterSecretStore
      target:
        name: fourdogs-central-secrets
        creationPolicy: Owner
      data:
        - secretKey: DATABASE_URL
          remoteRef:
            key: secret/terminus/fourdogs/central
            property: DATABASE_URL
        - secretKey: OAUTH_CLIENT_ID
          remoteRef:
            key: secret/terminus/fourdogs/central
            property: OAUTH_CLIENT_ID
        - secretKey: OAUTH_CLIENT_SECRET
          remoteRef:
            key: secret/terminus/fourdogs/central
            property: OAUTH_CLIENT_SECRET
        - secretKey: CORS_ALLOWED_ORIGIN
          remoteRef:
            key: secret/terminus/fourdogs/central
            property: CORS_ALLOWED_ORIGIN
    ```
  > Vault path `secret/terminus/fourdogs/central` per architecture Vault hierarchy. ClusterSecretStore name `vault-backend` — verify against existing ExternalSecrets in cluster if uncertain.
  - [ ] Commit: `feat(helm): add ExternalSecret for ESO/Vault-managed secrets (wave 0)`

- [ ] Task 4: Create `templates/ingressroute.yaml` (AC: 3)
  - [ ] Create `deploy/helm/fourdogs-central/templates/ingressroute.yaml`:
    ```yaml
    apiVersion: traefik.containo.us/v1alpha1
    kind: IngressRoute
    metadata:
      name: fourdogs-central
    spec:
      entryPoints:
        - websecure
      routes:
        - match: Host(`{{ .Values.ingress.host }}`)
          kind: Rule
          services:
            - name: {{ include "fourdogs-central.fullname" . }}
              port: {{ .Values.service.port }}
      tls:
        secretName: fourdogs-central-tls
    ---
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: fourdogs-central-tls
    spec:
      secretName: fourdogs-central-tls
      issuerRef:
        name: letsencrypt-prod  # match terminus cert-manager issuer name
        kind: ClusterIssuer
      dnsNames:
        - {{ .Values.ingress.host }}
    ```
  - [ ] Verify `ClusterIssuer` name against existing cert-manager config in cluster
  - [ ] Commit: `feat(helm): add Traefik IngressRoute with cert-manager TLS`

- [ ] Task 5: Create `templates/service.yaml` (AC: 4)
  - [ ] Create `deploy/helm/fourdogs-central/templates/service.yaml`:
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: {{ include "fourdogs-central.fullname" . }}
    spec:
      selector:
        {{- include "fourdogs-central.selectorLabels" . | nindent 4 }}
      ports:
        - protocol: TCP
          port: {{ .Values.service.port }}
          targetPort: {{ .Values.service.port }}
    ```
  - [ ] Commit: `feat(helm): add service`

- [ ] Task 6: Register in ArgoCD app-of-apps manifest (AC: 5, 6)
  - [ ] Locate the ArgoCD app-of-apps kustomization in `TargetProjects/terminus/infra`
  - [ ] Add fourdogs-central Application manifest pointing to `deploy/helm/fourdogs-central/`
  - [ ] Trigger ArgoCD sync (or wait for auto-sync)
  - [ ] Verify: `kubectl get pods -n fourdogs` — init container (`migrate`) completes, API pod is `Running`
  - [ ] Verify: `curl https://fourdogs.terminus.local/v1/health` returns `{"status":"ok"}` (AC: 5)
  - [ ] Verify: ArgoCD app shows `Synced` and `Healthy` (AC: 6)
  - [ ] Commit: `feat(argocd): register fourdogs-central application in app-of-apps`

- [ ] Task 7: Update sprint status
  - [ ] Set `1-5-helm-chart-and-argocd-deployment-to-k3s: done`
  - [ ] Set `epic-1: done`
  - [ ] Set `epic-2: in-progress`

## Dev Notes

- ARCH3: migrate init container pattern — the init container MUST exit 0 before the API pod is considered Ready; readinessProbe gates API traffic
- ARCH10: ESO secret names: `DATABASE_URL`, `OAUTH_CLIENT_ID`, `OAUTH_CLIENT_SECRET`, `CORS_ALLOWED_ORIGIN` — these exact names as Kubernetes secret keys
- The init container uses the same Docker image as the API container but overrides ENTRYPOINT with `/migrate up`
- **Vault path is `secret/terminus/fourdogs/central`** per architecture Vault hierarchy — not `kv/terminus/fourdogs-central` or other variations
- **`imagePullSecrets` is required** — without the GHCR ExternalSecret at wave -1 and `imagePullSecrets` in the Deployment, the pod will fail with `Init:ImagePullBackOff`. Never omit this for GHCR-hosted images.
- **Sync-wave order is contractual**: wave -1 (GHCR ExternalSecret) → wave 0 (service ExternalSecret) → wave 1+ (Deployment). See architecture.md — Consumer Service Deployment Pattern.
- If `terminus-infra-k3s` or `terminus-infra-secrets` is not live, STOP — this is a hard prerequisite
- Health check stub: `cmd/api/main.go` must at minimum serve `/v1/health` → `{"status":"ok"}` at this point (the full chi router wired in Story 3.1 replaces this)
  - For Story 1.5 only, you can add a minimal HTTP server that handles `/v1/health` before Story 3.1 is complete

### Project Structure Notes

- `deploy/helm/fourdogs-central/` — Helm chart directory
- ArgoCD Application manifest location: determined by your app-of-apps convention in terminus-infra
- `fourdogs-central-secrets` — the Kubernetes Secret name created by ESO; must match the `envFrom.secretRef.name` in Deployment

### References

- [Source: phases/techplan/architecture.md — Deployment section, ArgoCD, ESO]
- [Source: phases/devproposal/epics.md — Story 1.5 ACs, ARCH3, ARCH10, NFR7]
- [Source: phases/businessplan/prd.md — FR26 (k3s deployment), FR9 (health check)]

## Dev Agent Record

### Agent Model Used

GitHub Copilot / Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
