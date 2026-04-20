# Story: transactions-2-01

## Context

**Feature:** transactions
**Sprint:** 2 — Deploy & Validate
**Priority:** high
**Estimate:** M

## User Story

As an operator, I want a Helm chart and ArgoCD Application for `fourdogs-etailpet-trigger`, so that
the worker can be deployed to k3s via GitOps exactly like other fourdogs services.

## Acceptance Criteria

- [ ] Helm chart at `deploy/helm/fourdogs-etailpet-trigger/` in `fourdogs-central` repo
- [ ] Chart includes: `Deployment` (replicas: 1, no HPA), `ServiceAccount`, env var wiring from `Secret`
- [ ] `TRIGGER_ENABLED` defaults to `false` in `values.yaml` (safe default; enables without rollback risk)
- [ ] `TRIGGER_INTERVAL_HOURS` configurable in `values.yaml`; default: `24`
- [ ] Docker image built from `cmd/etailpet-trigger/`; pushed to GHCR as `ghcr.io/fourdogs/etailpet-trigger:{tag}`; CI step added to existing workflow
- [ ] ArgoCD Application manifest at `argocd/fourdogs-etailpet-trigger.yaml` in the platform GitOps repo (or equivalent location per platform convention)
- [ ] ArgoCD Application references `fourdogs` project; namespace: `fourdogs-central` (same as other fourdogs services)
- [ ] Deployment has a liveness probe defined (e.g. exec `true` or a `/healthz` HTTP endpoint if added; confirm against emailfetcher pattern)
- [ ] `helm lint deploy/helm/fourdogs-etailpet-trigger/` passes with no warnings
- [ ] `helm template` output reviewed and confirmed correct before ArgoCD sync

## Technical Notes

**Helm chart minimum structure:**
```
deploy/helm/fourdogs-etailpet-trigger/
├── Chart.yaml          — name: fourdogs-etailpet-trigger, version: 0.1.0, appVersion: "1.0.0"
├── values.yaml         — image.tag, triggerEnabled: false, triggerIntervalHours: 24
└── templates/
    ├── deployment.yaml
    └── serviceaccount.yaml
```

**Deployment spec highlights:**
```yaml
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: etailpet-trigger
          image: ghcr.io/fourdogs/etailpet-trigger:{{ .Values.image.tag }}
          env:
            - name: TRIGGER_ENABLED
              value: "{{ .Values.triggerEnabled }}"
            - name: TRIGGER_INTERVAL_HOURS
              value: "{{ .Values.triggerIntervalHours }}"
            - name: ETAILPET_API_KEY
              valueFrom:
                secretKeyRef:
                  name: fourdogs-etailpet-api-secrets
                  key: ETAILPET_API_KEY
            - name: ETAILPET_TRIGGER_URL
              valueFrom:
                secretKeyRef:
                  name: fourdogs-etailpet-api-secrets
                  key: ETAILPET_TRIGGER_URL
      serviceAccountName: fourdogs-etailpet-trigger
```

Follow the emailfetcher Helm chart as the canonical reference for resource limits, annotations,
and label conventions.

**Image build CI step:** Add a `docker build` + `docker push` step to the GitHub Actions workflow
for the `cmd/etailpet-trigger` binary, conditional on the branch or tag pattern used for other
fourdogs services.

## Dependencies

**Blocked by:** transactions-1-02 (binary must be implemented and compilable)
**Blocks:** transactions-2-02 (ESO ExternalSecret references namespace created by chart)

## Definition of Done

- [ ] Code compiles with no errors or warnings (`go build ./...`)
- [ ] `go test ./...` passes
- [ ] `go vet ./...` passes
- [ ] No credentials, tokens, or secrets in code or committed files
- [ ] Structured log events fire correctly for the story's scope
- [ ] Story acceptance criteria verified by author
- [ ] PR reviewed (or self-reviewed for solo work) and merged to `transactions` branch

## Open Questions

- Confirm target namespace for ArgoCD Application: `fourdogs-central` or a dedicated namespace?
  (Assumption: `fourdogs-central`, consistent with other fourdogs services.)
