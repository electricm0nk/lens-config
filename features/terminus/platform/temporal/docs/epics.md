# Epics — terminus-platform-temporal

**Track:** tech-change
**Source:** Architecture Decision Document + Adversarial Review (PASS_WITH_NOTES)
**Sprint Planning by:** Bob (Scrum Master)
**Date:** 2026-03-31
**Phase:** SprintPlan (sprintplan, large audience)

---

> **Note:** This initiative follows the tech-change track. Epics and stories are derived directly
> from the architecture document and adversarial review findings. There is no preceding devproposal
> phase. Stories are ordered for safe sequential execution.
>
> **Adversarial Review Blockers Addressed:**
> - B1 (ESO ClusterSecretStore name) → resolved in Story 2.2 AC
> - B2 (ArgoCD Application manifest location) → resolved in Story 2.3 AC
> - B3 (Story decomposition) → applied throughout — server and worker are decomposed into atomic stories
>
> **N-Notes Addressed:**
> N1 → Story 3.2 AC | N2 → Story 4.1 | N3 → Story 1.2 AC | N4 → Story 4.1 dependency note
> N5 → Story 3.1 AC | N6 → Story 1.1 AC | N7 → Story 3.1 AC | N8 → Stories 2.1, 2.2 boundary

---

## Dependency Map

```
Story 1.1 (monorepo scaffold)
  └── Story 1.2 (DB provisioning)           ← independent of 1.1, can parallelize
        └── Story 2.1 (namespace + RBAC)
              └── Story 2.2 (ESO ExternalSecrets)
                    └── Story 2.3 (Temporal server Helm + ArgoCD)
                          └── Story 2.4 (Traefik IngressRoute + cert-manager)
                                └── Story 3.1 (Worker Helm + ArgoCD)
                                      └── Story 3.2 (TypeScript worker scaffold)
                                            └── Story 3.3 (CI image pipeline)
                                                  └── Story 4.1 (Smoke test)
```

Stories 1.1 and 1.2 can execute in parallel. All others are sequentially dependent.

---

## Epic 1: Platform Foundation

**Goal:** Establish the `terminus.platform` monorepo scaffold and provision the PostgreSQL databases required by Temporal. These are prerequisite stories — all subsequent epics depend on both completing.

**Scope:**
- Story Zero A: Initialize terminus.platform monorepo with npm workspaces, shared/types, base CI
- Story Zero B: Patroni DB provisioning (temporal + temporal_visibility databases, dedicated user, Vault secrets)

---

### Story 1.1: Initialize terminus.platform Monorepo Scaffold

**Priority:** BLOCKER — must be completed before any TypeScript implementation story
**Source:** Architecture Step 6 (Project Structure), N6 (DoD gap)
**Depends-on:** Nothing (no upstream dependency)

**Context:**
`terminus.platform` is currently a stub repo with no `package.json`, `tsconfig.json`, `shared/` workspaces, or `.github/workflows/`. This story establishes the full monorepo structure defined in the architecture. All subsequent TypeScript stories depend on the workspace being bootstrapped.

**Acceptance Criteria:**
- [ ] `package.json` with `workspaces: ["services/*", "shared/*"]` at repo root
- [ ] Root `tsconfig.json` with `strict: true`; all services extend it
- [ ] `shared/types/` package: `module.ts` (Module interface, ModulePayload, RunContext), `index.ts`, `package.json`, `tsconfig.json`
- [ ] `shared/lib/` package: `logger.ts` (pino factory), `errors.ts` (ApplicationFailure helpers), `package.json`, `tsconfig.json`
- [ ] `services/temporal/` directory scaffolded with `package.json`, `tsconfig.json`, `vitest.config.ts`, `src/` (empty: `worker.ts`, `workflows/`, `activities/`, `lib/`)
- [ ] `.github/workflows/ci.yml` — runs tests for all changed services on PR (skeleton, can be non-functional initially)
- [ ] `.github/workflows/publish-images.yml` — skeleton CI workflow file (must be syntactically valid YAML)
- [ ] `npm install` succeeds cleanly at repo root (no errors)
- [ ] `shared/types` TypeScript compiles clean (`npx tsc --noEmit` in `shared/types/`)
- [ ] Prometheus and Grafana stub directories (`services/grafana/`, `services/prometheus/`) are **NOT** created — per A3 from adversarial review, these belong to future initiatives

**Implementation Notes:**
- Do not create any workflow or activity implementations — this story is scaffold only
- `shared/types/src/module.ts` must define: `Module`, `ModulePayload<S>` (with required `schema` field), `RunContext`
- Pino logger factory in `shared/lib` sets `service: terminus-platform-worker` by default
- Reference: architecture Step 6 complete directory tree

---

### Story 1.2: Patroni DB Provisioning for Temporal

**Priority:** BLOCKER — must complete before Temporal Helm chart can run schema migrations
**Source:** Architecture Step 2 (DB provisioning in-scope), N3 (mechanism undefined)
**Depends-on:** Patroni cluster operational (`terminus-infra-postgres` prerequisite met)

**Context:**
The `temporalio/temporal` Helm chart runs schema migration Jobs (`admintools`) against PostgreSQL on install. These Jobs require the `temporal` and `temporal_visibility` databases and a dedicated `temporal` DB user to exist before the chart is applied. This is a Patroni admin task — SQL scripts run against the Patroni primary. No Kubernetes manifests needed.

**Acceptance Criteria:**
- [ ] SQL provisioning script committed to `docs/scripts/db-provision-temporal.sql` in `terminus.platform`
- [ ] Script creates `temporal` database
- [ ] Script creates `temporal_visibility` database
- [ ] Script creates `temporal` DB user with password reference (no plaintext password in script)
- [ ] Script grants `temporal` user full privileges on both databases
- [ ] Vault secret at `secret/terminus/default/temporal/db-password` created with the generated password
- [ ] Vault secret at `secret/terminus/default/temporal/visibility-db-password` created (same or separate user — document decision in script header)
- [ ] Script is idempotent (uses `CREATE DATABASE IF NOT EXISTS`, `CREATE USER IF NOT EXISTS`)
- [ ] Runbook for running the script documented in script header comments: `psql -h 10.0.0.56 -U admin -f db-provision-temporal.sql`

**Implementation Notes:**
- Connect to Patroni primary at `10.0.0.56:5432`
- Admin credentials retrieved from Vault at `secret/terminus/default/patroni/admin-password` (established convention)
- The `temporal_visibility` DB can use the same `temporal` user — simpler for homelab, aligns with architecture (two Vault paths, both for `temporal` user)
- Do NOT include plaintext passwords anywhere in the script or commit — reference only

---

## Epic 2: Temporal Server Deployment

**Goal:** Deploy the Temporal server into the k3s cluster via ArgoCD App-of-Apps. This epic covers the full server-side delivery: namespace, RBAC, Vault secrets, ESO ExternalSecrets, Helm configuration, ArgoCD Application, and Traefik ingress for the Web UI.

**Scope:**
- k8s namespace and RBAC isolation (Story 2.1)
- ESO ExternalSecrets for DB credentials (Story 2.2) — resolves B1 (ClusterSecretStore name)
- Temporal server Helm values + ArgoCD Application (Story 2.3) — resolves B2 (ArgoCD file location)
- Traefik IngressRoute + cert-manager TLS (Story 2.4)

---

### Story 2.1: Temporal k8s Namespace + RBAC

**Priority:** High — required foundation for all Temporal k8s resources
**Source:** Architecture Step 6 (`k8s/namespace.yaml`, `k8s/rbac-worker.yaml`), N8 (boundary separation)
**Depends-on:** Story 1.2 (Vault secrets exist before ESO can sync)

**Context:**
The `temporal` k8s namespace isolates all Temporal server and worker resources. The worker ServiceAccount and ClusterRole/binding are committed here as raw manifests in `k8s/` per the architecture anti-pattern rule (ESO and RBAC only in `k8s/`). ArgoCD sync wave 1 governs these — they must exist before ESO ExternalSecrets sync.

**Acceptance Criteria:**
- [ ] `k8s/namespace.yaml` created: `apiVersion: v1, kind: Namespace, metadata: {name: temporal}`
- [ ] `k8s/rbac-worker.yaml` created with:
  - `ServiceAccount` in `temporal` namespace: `temporal-worker`
  - `ClusterRole` or `Role`: read access to `temporal` namespace secrets (for worker to read ESO-synced secrets)
  - `RoleBinding` binding `temporal-worker` ServiceAccount to the Role
- [ ] Both files have ArgoCD sync wave annotation: `argocd.argoproj.io/sync-wave: "1"`
- [ ] Files pass `kubectl apply --dry-run=client -f <file>` against the k3s cluster
- [ ] `k8s/` directory contains only these two files — no Helm overrides or application configs

**Implementation Notes:**
- Sync wave 1 ensures namespace and RBAC exist before ESO ExternalSecret resources try to sync
- ClusterRole vs Role: prefer `Role` scoped to `temporal` namespace (principle of least privilege)
- The ServiceAccount `temporal-worker` will be referenced by the worker Deployment in Story 3.1

---

### Story 2.2: ESO ExternalSecrets for Temporal DB Credentials

**Priority:** High — Temporal Helm chart requires k8s Secrets to exist before pod startup
**Source:** Architecture Step 4 (secrets delivery via ESO), B1 (ClusterSecretStore name must be resolved)
**Depends-on:** Story 2.1 (namespace exists), Story 1.2 (Vault secrets exist)

**Context:**
Two ExternalSecret resources pull Vault credentials into the `temporal` namespace: one for the persistence DB password, one for the visibility DB password. The ClusterSecretStore name is not in the architecture document (adversarial finding B1) — implementing agent must resolve from domain architecture before writing these manifests.

**Acceptance Criteria:**
- [ ] **Resolve B1 (blocking):** Look up the ClusterSecretStore name from domain architecture or the crossplane initiative's ESO configuration. Document the resolved name in this story's implementation notes before writing manifests.
- [ ] `k8s/external-secret-db.yaml` created: `ExternalSecret` named `temporal-db-credentials` in `temporal` namespace
  - `secretStoreRef.kind: ClusterSecretStore`, `name: {resolved-name}`
  - `remoteRef.key: secret/terminus/default/temporal/db-password`
  - `target.name: temporal-db-credentials`
  - `refreshInterval: 1h`
- [ ] `k8s/external-secret-visibility-db.yaml` created: `ExternalSecret` named `temporal-visibility-db-credentials`
  - Same store, `remoteRef.key: secret/terminus/default/temporal/visibility-db-password`
  - `target.name: temporal-visibility-db-credentials`
  - `refreshInterval: 1h`
- [ ] Both ExternalSecrets have sync wave annotation: `argocd.argoproj.io/sync-wave: "1"` (same wave as namespace — ESO syncs early)
- [ ] Both files pass YAML linting
- [ ] After applying to cluster, verify k8s Secrets are created in `temporal` namespace

**Implementation Notes:**
- The crossplane initiative used the established ClusterSecretStore — reference `docs/terminus/infra/crossplane/architecture.md` or the deployed ESO configuration to find the store name
- ESO ExternalSecret API group: `external-secrets.io/v1beta1`
- The resolved ClusterSecretStore name must be documented in a comment at the top of each ExternalSecret file

---

### Story 2.3: Temporal Server Helm Values + ArgoCD Application

**Priority:** High — core server deployment
**Source:** Architecture Steps 3, 4, 5 (Helm chart, ArgoCD App-of-Apps), B2 (Application file location)
**Depends-on:** Story 2.2 (DB credentials exist as k8s Secrets), Story 1.2 (DB exists)

**Context:**
This story deploys the `temporalio/temporal` Helm chart (`1.0.0-rc.3`) via ArgoCD. The Helm values disable Cassandra and Elasticsearch, configure external PostgreSQL, and reference ESO-synced Secrets. Adversarial finding B2 flagged that the ArgoCD Application manifest location was unspecified — implementing agent must follow the App-of-Apps pattern established by crossplane.

**Acceptance Criteria:**
- [ ] **Resolve B2 (blocking):** Identify the ArgoCD App-of-Apps root repo and directory where Application CRs live. Follow the same pattern as `terminus-infra-crossplane`. Document the resolved location in this story's implementation notes before writing the Application manifest.
- [ ] `helm/temporal-server/` directory created (or values file at path matching App-of-Apps convention — see B2 resolution)
- [ ] Helm `values.yaml` for Temporal server includes:
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
  - `server.metrics.prometheus.listenAddress: "0.0.0.0:9090"` (Prometheus scrape endpoint)
- [ ] ArgoCD `Application.yaml` at the resolved location:
  - `spec.source.repoURL: https://go.temporal.io/helm-charts`
  - `spec.source.chart: temporal`
  - `spec.source.targetRevision: 1.0.0-rc.3`
  - `spec.destination.namespace: temporal`
  - `spec.syncPolicy.automated: {prune: true, selfHeal: true}`
  - `argocd.argoproj.io/sync-wave: "2"` (after ESO secrets at wave 1)
- [ ] Values file has comment noting Patroni IP 10.0.0.56 is intentionally hard-coded (stable bare metal IP)
- [ ] Schema migration Job (`admintools`) runs successfully on first sync (confirms DB credentials and schema bootstrap work)
- [ ] All Temporal server pods reach `Running` state in `temporal` namespace

**Implementation Notes:**
- Cassandra is bundled by default and **must** be explicitly disabled
- Elasticsearch is bundled by default and **must** be explicitly disabled
- Schema migration Job uses `admintools` image — confirm it runs before frontend pod starts
- Sync wave `"2"` — applied string-quoted per Kubernetes annotation conventions

---

### Story 2.4: Traefik IngressRoute + cert-manager Certificate for Temporal Web UI

**Priority:** Medium — UI access is not required for worker operation
**Source:** Architecture Step 4 (ingress hostname `temporal-ui.trantor.internal`, TLS via cert-manager Vault PKI)
**Depends-on:** Story 2.3 (Temporal server running, Web UI pod healthy)

**Context:**
The Temporal Web UI runs on port 8080 and is only accessible within the cluster. This story exposes it via Traefik on `temporal-ui.trantor.internal` with TLS from the cert-manager Vault PKI issuer (consistent with all other k3s services).

**Acceptance Criteria:**
- [ ] `k8s/ingress-route.yaml` created (or Helm chart IngressRoute template — match existing terminus pattern):
  - `kind: IngressRoute` (Traefik CRD)
  - `spec.routes[0].match: "Host(\`temporal-ui.trantor.internal\`)"` 
  - `spec.routes[0].services[0].name: temporal-web` (Temporal chart service name)
  - `spec.routes[0].services[0].port: 8080`
  - `spec.tls.secretName: temporal-ui-tls`
- [ ] `k8s/certificate.yaml` created:
  - `kind: Certificate`
  - `spec.dnsNames: [temporal-ui.trantor.internal]`
  - `spec.issuerRef.name: {vault-pki-issuer}` (follow existing terminus cert pattern)
  - `spec.issuerRef.kind: ClusterIssuer`
  - `spec.secretName: temporal-ui-tls`
- [ ] Both resources in `temporal` namespace
- [ ] `https://temporal-ui.trantor.internal` accessible in browser (responds with Temporal Web UI)
- [ ] Certificate is valid and trusted (issued by Vault PKI)
- [ ] Documents the Vault PKI ClusterIssuer name used (follow pattern from existing k3s services)

**Implementation Notes:**
- The cert-manager ClusterIssuer name follows the established terminus pattern — reference existing cert resources (e.g., from the k3s infrastructure initiative) rather than inventing a new name
- IngressRoute is a Traefik CRD (group: `traefik.io`), not a standard k8s Ingress
- Story scope: IngressRoute and Certificate only — do not add auth or rate limiting

---

## Epic 3: Temporal Worker Deployment

**Goal:** Deploy the TypeScript Temporal worker to k3s and establish the CI pipeline for image builds. The worker connects to the Temporal server and registers workflows and activities for the `terminus-platform` task queue.

**Scope:**
- Worker Helm chart + ArgoCD Application (Story 3.1)
- TypeScript worker scaffold: DailyBriefingWorkflow skeleton + activities (Story 3.2)
- CI image build pipeline — GHCR (Story 3.3)

---

### Story 3.1: Worker Helm Chart + ArgoCD Application

**Priority:** High — required for worker deployment to k3s
**Source:** Architecture Step 6 (`helm/temporal-worker/`), N5 (restart behavior), N7 (image bootstrap)
**Depends-on:** Story 1.1 (monorepo scaffold exists), Story 2.3 (Temporal server running)

**Context:**
The Temporal worker is deployed as a k8s Deployment managed by a custom Helm chart in `terminus.platform/helm/temporal-worker/`. It runs at `replicas: 1` and uses the `temporal-worker` ServiceAccount created in Story 2.1. This story creates the Helm chart, values.yaml, and ArgoCD Application for the worker.

**Acceptance Criteria:**
- [ ] `helm/temporal-worker/Chart.yaml` created with appropriate name and version
- [ ] `helm/temporal-worker/values.yaml` includes:
  - `image.repository: ghcr.io/electricm0nk/terminus-platform-worker`
  - `image.tag: latest` (bootstrap placeholder — Story 3.3 pins this to digest)
  - `image.pullPolicy: Always` (since tag is mutable during bootstrap)
  - `replicaCount: 1`
  - `resources.requests: {cpu: 100m, memory: 256Mi}`
  - `resources.limits: {cpu: 500m, memory: 512Mi}`
  - `temporal.address: temporal-frontend.temporal.svc.cluster.local:7233`
  - `temporal.namespace: default`
  - `temporal.taskQueue: terminus-platform`
  - Comment acknowledging `replicas: 1` rolling restart drops in-flight work — acceptable for homelab; Temporal retry policy covers retries
- [ ] `helm/temporal-worker/templates/deployment.yaml`: references `temporal-worker` ServiceAccount, mounts no credentials directly (all via ESO-synced Secrets if needed)
- [ ] `helm/temporal-worker/templates/serviceaccount.yaml`: `serviceAccountName: temporal-worker` (references SA from Story 2.1)
- [ ] ArgoCD `Application.yaml` at the resolved App-of-Apps location:
  - Points to `terminus.platform` repo, `helm/temporal-worker/` path
  - `spec.destination.namespace: temporal`
  - `argocd.argoproj.io/sync-wave: "3"` (after Temporal server at wave 2)
  - `spec.syncPolicy.automated: {prune: true, selfHeal: true}`
- [ ] **Image bootstrap:** Document in story completion notes that first deploy requires a manual image push to GHCR before the Deployment can reach `Running`. Placeholder strategy: use `ghcr.io/electricm0nk/terminus-platform-worker:bootstrap` tag with a minimal Node.js image; replace with real image after Story 3.3.
- [ ] Worker pod reaches `Running` state with the bootstrap image (may log connection errors to Temporal — expected until Stories 3.2/3.3 produce a real worker)

**Implementation Notes:**
- Sync wave `"3"` — worker must not start before Temporal server is healthy (wave 2)
- `serviceAccountName` references the SA from Story 2.1, not `default`
- No secrets in the Deployment spec — any future worker credentials come via ESO-synced Secrets mounted as env vars

---

### Story 3.2: TypeScript Worker Scaffold — DailyBriefingWorkflow

**Priority:** High — required to register workflows/activities on the `terminus-platform` task queue
**Source:** Architecture Steps 4, 5, 6 (worker entrypoint, workflow, activity structure), N1 (health probe)
**Depends-on:** Story 1.1 (monorepo scaffold exists)

**Context:**
This story implements the Temporal worker entrypoint and the `DailyBriefingWorkflow` skeleton with placeholder activities. The worker must successfully connect to Temporal, register the workflow, and process tasks on the `terminus-platform` queue. Full activity implementation belongs to the `terminus-agent-dailybriefing` initiative — this story creates the structure and stubs.

**Acceptance Criteria:**
- [ ] `services/temporal/src/worker.ts`: connects to `temporal-frontend.temporal.svc.cluster.local:7233`, registers `DailyBriefingWorkflow` and `briefingActivities`, polles `terminus-platform` task queue
- [ ] `services/temporal/src/workflows/dailyBriefingWorkflow.ts`: skeleton workflow function — `async function dailyBriefingWorkflow(payload: ModulePayload<DailyBriefingSchema>): Promise<void>` — calls at least one activity
- [ ] `services/temporal/src/workflows/dailyBriefingWorkflow.test.ts`: co-located test; mocks activity calls; TDD scaffolded before implementation
- [ ] `services/temporal/src/activities/briefingActivities.ts`: one placeholder activity (`fetchBriefingData`) — returns stub `ModulePayload<BriefingDataSchema>` with `schema` declared
- [ ] `services/temporal/src/activities/briefingActivities.test.ts`: co-located test, 80%+ line coverage
- [ ] `services/temporal/src/lib/logger.ts` and `src/lib/errors.ts`: pino logger factory + ApplicationFailure helpers
- [ ] All workflow functions use `@temporalio/sdk` — no `Date.now()`, `Math.random()`, or non-deterministic APIs
- [ ] All activity errors re-thrown as `ApplicationFailure.nonRetryable()` or `ApplicationFailure` (never raw `Error`)
- [ ] `npm run test` (or `npx vitest run`) passes with 80%+ line coverage and 100% on `ModulePayload` schema validation
- [ ] **N1 (health probe):** Define and document the health probe command in `services/temporal/src/lib/health.ts` — a script that attempts a gRPC probe to `temporal-frontend.temporal.svc.cluster.local:7233` and exits 0/1. The worker Helm chart's `livenessProbe` references this script.
- [ ] `services/temporal/package.json` declares workspace dependency on `shared/types` and `shared/lib`

**Implementation Notes:**
- `pino` logger — every module uses `createLogger({ module: 'module-name' })`
- `ModulePayload.schema` is required on every payload type — no untyped payloads
- `worker.ts` must NOT call `Date.now()` or non-deterministic APIs — only the activities layer does I/O
- Workflow determinism: all non-deterministic operations (HTTP calls, DB reads, timestamp generation) belong in activities
- TDD: write tests first, then implement to pass them

---

### Story 3.3: CI Image Build Pipeline — GHCR

**Priority:** Medium — required to produce a deployable worker image
**Source:** Architecture Step 6 (`.github/workflows/publish-images.yml`), N7 (image bootstrap gap)
**Depends-on:** Story 1.1 (CI skeleton exists), Story 3.2 (worker source code exists)

**Context:**
The worker image must be built and pushed to GHCR (`ghcr.io/electricm0nk/terminus-platform-worker`) on merge to the default branch. This story implements the full `publish-images.yml` CI workflow. Until this runs successfully, the worker Deployment uses the bootstrap image from Story 3.1.

**Acceptance Criteria:**
- [ ] `.github/workflows/publish-images.yml` fully implemented:
  - Trigger: `push` to `main` branch, `paths: ["services/temporal/**"]`
  - Build: `docker build -f services/temporal/Dockerfile -t ghcr.io/electricm0nk/terminus-platform-worker:${{ github.sha }} .`
  - Push: auth to GHCR via `GITHUB_TOKEN`, push both `:{sha}` and `:latest` tags
  - On success: output image digest
- [ ] `services/temporal/Dockerfile` created:
  - Multi-stage: build stage compiles TypeScript, runtime stage runs compiled JS
  - Base image: `node:lts-alpine` (LTS + alpine for minimal size)
  - Installs only production dependencies in runtime stage (`npm ci --omit=dev`)
  - Non-root user in runtime stage
  - Entrypoint: `node dist/worker.js`
- [ ] CI workflow runs successfully on merge to `main` and pushes image to GHCR
- [ ] Worker deployment in Story 3.1 updated to reference image digest (replace bootstrap placeholder)
- [ ] **N7 bootstrap sequence:** After CI publishes the first real image, update the ArgoCD Application to pin `image.tag` to the first successful build digest. Document this one-time step in Story 3.1's implementation notes.

**Implementation Notes:**
- GHCR authentication uses the workflow's built-in `GITHUB_TOKEN` — no extra credentials needed
- Dockerfile multi-stage: avoids shipping devDependencies and TypeScript compiler in production image
- After first successful CI build, update `helm/temporal-worker/values.yaml` to pin the image digest

---

## Epic 4: Integration Verification

**Goal:** Verify the complete end-to-end deployment: Temporal server is operational, worker is connected, and a `DailyBriefingWorkflow` can be triggered and observed through the visibility store.

---

### Story 4.1: Smoke Test — DailyBriefingWorkflow End-to-End

**Priority:** High — confirms system is operational before handing off to dailybriefing initiative
**Source:** N2 (no smoke test defined), N4 (dailybriefing sequencing dependency)
**Depends-on:** Stories 2.3, 3.1, 3.2, 3.3 all complete and worker deployed with real image

**Context:**
After all deployment stories land, there must be a verifiable end state for "Temporal is operational." This story defines and executes that verification. Without this gate, the `terminus-agent-dailybriefing` initiative cannot safely begin implementation stories that depend on a running Temporal instance.

**Acceptance Criteria:**
- [ ] Using `tctl` or Temporal Web UI (`https://temporal-ui.trantor.internal`), trigger a `DailyBriefingWorkflow` execution on namespace `default`, task queue `terminus-platform`
- [ ] Workflow execution appears in the Temporal Web UI with `Running` → `Completed` status
- [ ] Workflow completion result is visible in the PostgreSQL visibility store (verified via Web UI search or `tctl workflow list`)
- [ ] Worker logs show `DailyBriefingWorkflow` activity execution with expected pino structured output (fields: `timestamp`, `level`, `service`, `module`, `message`, `context`)
- [ ] No errors in Temporal server `temporal-frontend` pod logs during workflow execution
- [ ] Smoke test results documented in `docs/terminus/platform/temporal/smoke-test-results.md`: timestamp, workflow ID, result, screenshot or log excerpt
- [ ] **N4 (dailybriefing sequencing):** Add note to `docs/terminus/platform/temporal/smoke-test-results.md`: "Temporal server + worker verified operational on [date]. terminus-agent-dailybriefing implementation stories may now proceed."

**Implementation Notes:**
- `tctl` is available in the `temporal-admintools` pod: `kubectl exec -it temporal-admintools-<pod> -n temporal -- tctl workflow run --tq terminus-platform --wt DailyBriefingWorkflow --et 60`
- Alternatively, use the Temporal Web UI's "Start Workflow" feature
- Web UI URL: `https://temporal-ui.trantor.internal`
- This story is a gate for the dailybriefing initiative — its completion should trigger a `@status` check on the dailybriefing initiative to see if it is blocked on this

---

## Sprint Backlog Summary

| Story | Title | Epic | Priority | Depends On |
|---|---|---|---|---|
| 1.1 | Initialize terminus.platform Monorepo Scaffold | Foundation | BLOCKER | — |
| 1.2 | Patroni DB Provisioning for Temporal | Foundation | BLOCKER | — |
| 2.1 | Temporal k8s Namespace + RBAC | Server | High | 1.2 |
| 2.2 | ESO ExternalSecrets for Temporal DB Credentials | Server | High | 2.1, 1.2 |
| 2.3 | Temporal Server Helm Values + ArgoCD Application | Server | High | 2.2, 1.2 |
| 2.4 | Traefik IngressRoute + cert-manager Certificate | Server | Medium | 2.3 |
| 3.1 | Worker Helm Chart + ArgoCD Application | Worker | High | 1.1, 2.3 |
| 3.2 | TypeScript Worker Scaffold — DailyBriefingWorkflow | Worker | High | 1.1 |
| 3.3 | CI Image Build Pipeline — GHCR | Worker | Medium | 1.1, 3.2 |
| 4.1 | Smoke Test — DailyBriefingWorkflow End-to-End | Verification | High | 2.3, 3.1, 3.2, 3.3 |

**Total stories:** 10
**Parallel start set:** 1.1 and 1.2 (no dependencies)
**Critical path:** 1.2 → 2.1 → 2.2 → 2.3 → 3.1 → 4.1
