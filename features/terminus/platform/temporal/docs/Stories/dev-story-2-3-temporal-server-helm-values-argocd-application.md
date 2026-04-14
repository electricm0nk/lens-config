# Story 2.3: Temporal Server Helm Values + ArgoCD Application

Status: done

## Story

As a platform developer,
I want the Temporal server deployed to k3s via the `temporalio/temporal` Helm chart managed by ArgoCD,
so that the Temporal server is operational with PostgreSQL persistence and automated schema migration on every sync.

## Acceptance Criteria

1. **Resolve B2 (blocking):** ArgoCD App-of-Apps root repo and directory identified by following the same pattern as `terminus-infra-crossplane`. Resolved location documented in implementation notes before writing the Application manifest.
2. `helm/temporal-server/` directory created (or path per B2 resolution matching App-of-Apps convention)
3. Helm `values.yaml` for Temporal server includes:
   - `cassandra.enabled: false`
   - `elasticsearch.enabled: false`
   - `server.config.persistence.default.driver: sql`
   - `server.config.persistence.default.sql.driver: postgres`
   - PostgreSQL host `10.0.0.56`, port `5432`, database `temporal`
   - References `temporal-db-credentials` Secret for DB password (not plaintext)
   - `server.config.persistence.visibility.sql.database: temporal_visibility`
   - References `temporal-visibility-db-credentials` Secret
   - `web.enabled: true`
   - `admintools.enabled: true` (for schema migration Job)
   - `server.metrics.prometheus.listenAddress: "0.0.0.0:9090"`
4. ArgoCD `Application.yaml` at the resolved location:
   - `spec.source.repoURL: https://go.temporal.io/helm-charts`
   - `spec.source.chart: temporal`
   - `spec.source.targetRevision: 1.0.0-rc.3`
   - `spec.destination.namespace: temporal`
   - `spec.syncPolicy.automated: {prune: true, selfHeal: true}`
   - `argocd.argoproj.io/sync-wave: "2"`
5. Values file has comment noting Patroni IP `10.0.0.56` is intentionally hard-coded (stable bare metal IP)
6. Schema migration Job (`admintools`) runs successfully on first sync
7. All Temporal server pods reach `Running` state in `temporal` namespace

## Tasks / Subtasks

- [ ] Task 1: Resolve B2 — App-of-Apps Application manifest location (AC: 1)
  - [ ] Examine `terminus-infra-crossplane` ArgoCD Application pattern (check `terminus.infra` repo)
  - [ ] Identify the App-of-Apps root Application and directory where Application CRs live
  - [ ] Document the resolved path in implementation notes (e.g., `terminus.infra/argocd/applications/temporal-server.yaml`)
  - [ ] Confirm the pattern: does `terminus.platform` own its own Application CRs, or does `terminus.infra` manage them?

- [ ] Task 2: Create Helm values file (AC: 2, 3, 5)
  - [ ] Create `helm/temporal-server/values.yaml` in `terminus.platform`
  - [ ] Disable Cassandra: `cassandra.enabled: false`
  - [ ] Disable Elasticsearch: `elasticsearch.enabled: false`
  - [ ] Configure PostgreSQL persistence: host `10.0.0.56`, port `5432`, `driver: sql`, `sql.driver: postgres`
  - [ ] Add comment: `# Patroni primary IP — intentionally hard-coded (stable bare metal host)`
  - [ ] Reference ESO-synced Secrets for DB passwords (not plaintext values)
  - [ ] Enable web UI: `web.enabled: true`
  - [ ] Enable admintools: `admintools.enabled: true`
  - [ ] Add Prometheus scrape endpoint: `server.metrics.prometheus.listenAddress: "0.0.0.0:9090"`
  - [ ] Commit: `feat(helm): add temporal-server Helm values`

- [ ] Task 3: Create ArgoCD Application manifest (AC: 4)
  - [ ] Create Application CR at the resolved App-of-Apps location
  - [ ] Set `spec.source.repoURL: https://go.temporal.io/helm-charts`
  - [ ] Set `spec.source.chart: temporal`, `targetRevision: 1.0.0-rc.3`
  - [ ] Set `spec.destination.namespace: temporal`
  - [ ] Enable automated sync: `prune: true`, `selfHeal: true`
  - [ ] Set `argocd.argoproj.io/sync-wave: "2"` (string-quoted)
  - [ ] Reference `helm/temporal-server/values.yaml` for overrides
  - [ ] Commit: `feat(argocd): add Temporal server Application`

- [ ] Task 4: Verify deployment (AC: 6, 7)
  - [ ] Trigger ArgoCD sync
  - [ ] Confirm admintools schema migration Job completes: `kubectl get jobs -n temporal`
  - [ ] Confirm all pods Running: `kubectl get pods -n temporal`
  - [ ] Confirm frontend, history, matching, worker components all healthy

## Dev Notes

### Target Repo Path
```
TargetProjects/terminus/platform/terminus.platform/    (Helm values)
TargetProjects/terminus/infra/terminus.infra/           (ArgoCD Application — per B2 resolution)
```

### B2 Resolution
Adversarial finding B2: ArgoCD Application manifest location was unspecified. Resolution: follow the pattern established by `terminus-infra-crossplane`. The implementing agent must examine how crossplane's ArgoCD Application is structured and mirror that pattern for the Temporal server Application.

### Helm Chart
- Chart: `temporalio/temporal`
- Version: `1.0.0-rc.3`
- Repo: `https://go.temporal.io/helm-charts`
- Default chart bundles Cassandra and Elasticsearch — both **must** be explicitly disabled

### Sync Waves
- Wave `"1"`: namespace, RBAC, ESO ExternalSecrets (Stories 2.1, 2.2)
- Wave `"2"`: Temporal server (this story)
- Wave `"3"`: Temporal worker (Story 3.1)

### Patroni IP
`10.0.0.56` is intentionally hard-coded. Patroni primary is a stable bare metal host — no DNS required for homelab.

### Constitutional Override — Develop-First Integration
Implementation work lands on `develop` in `terminus.platform`; follow the active repo convention for `terminus.infra` Application CRs.

### Not In Scope
- Traefik IngressRoute for Web UI (Story 2.4)
- Worker deployment (Story 3.1)

## Pre-Implementation Constitution Gates

| Gate | Article | Result |
|------|---------|--------|
| Track declared | Org Art. 1 | ✅ PASS — `tech-change` |
| Architecture documented | Org Art. 3 | ✅ PASS — `docs/terminus/platform/temporal/architecture.md` |
| No confidential data | Org Art. 4 | ✅ PASS — no plaintext credentials in values or manifests |
| Git discipline | Org Art. 5 | ✅ PASS — develop-first integration authorized by terminus Art. 5 |
| Security first | Org Art. 9 | ✅ PASS — DB credentials via ESO k8s Secrets, not plaintext |
| Repo as source of truth | Org Art. 10 | ✅ PASS — all context from committed docs |
| Develop-first integration | Terminus Art. 5 | ✅ OVERRIDE ACTIVE — land implementation work on `develop` in `terminus.platform` |

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (copilot)

### Debug Log References

### Completion Notes List

### File List
