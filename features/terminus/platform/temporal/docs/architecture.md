---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
lastStep: 8
status: complete
completedAt: '2026-03-31'
inputDocuments:
  - docs/terminus/architecture.md
  - Docs/terminus/platform/project-context.md
  - _bmad-output/lens-work/initiatives/terminus/platform/temporal.yaml
  - TargetProjects/terminus/platform/terminus.platform/README.md
workflowType: architecture
project_name: terminus-platform-temporal
user_name: Todd
date: '2026-03-30'
---

# Architecture Decision Document — Temporal on terminus-platform

_This document builds collaboratively through step-by-step discovery. Sections are appended as we work through each architectural decision together._

---

## Project Context (Step 2)

**Initiative:** terminus-platform-temporal
**Track:** tech-change
**Target repo:** terminus.platform
**Layer:** terminus-platform (consumer of infra — not infra itself)

### Confirmed Context Decisions

| Decision | Choice | Notes |
|---|---|---|
| Layer classification | `terminus-platform` | Deploys from `terminus.platform` via Helm + ArgoCD |
| Environment strategy | Single k3s cluster, single env | No multi-env complexity; public cloud if scaling needed later |
| Temporal namespace | `default` | Single namespace sufficient for single environment |
| DB provisioning | In-scope for this initiative | New stories: create `temporal` DB + dedicated user on Patroni |
| DB schema bootstrap | Kubernetes Job (in-band) | Runs as part of Helm install sequence |
| Secrets management | Vault KV v2 + ESO | No hardcoded credentials; no secrets in git |
| Worker runtime | TypeScript + `@temporalio/worker` | k8s Deployment |
| Deployment method | Helm (required) + ArgoCD GitOps | All k8s workloads use this pattern |

### Dependency Map

- **Prerequisite:** `terminus-infra-k3s` — cluster must be running
- **Prerequisite:** `terminus-infra-postgres` — Patroni cluster must be running; DB provisioning is in-scope
- **Prerequisite:** `terminus-infra-secrets` — Vault must be running; ESO must be installed
- **In-scope:** PostgreSQL database + user creation for Temporal
- **In-scope:** Vault secret paths for Temporal credentials
- **Out of scope:** k3s cluster management, Patroni management, Vault management

---

## Core Architectural Decisions (Step 4)

### Decision Priority Analysis

**Critical Decisions (Block Implementation):**
- PostgreSQL DB + user provisioning on Patroni (prerequisite story — must complete first)
- Vault secret paths established before Helm install
- Helm chart + values structure (external DB, disabled Cassandra/ES)

**Important Decisions (Shape Architecture):**
- Single k8s namespace `temporal` for all Temporal components
- Worker connects to Temporal frontend via in-cluster DNS (no external exposure)
- ArgoCD sync waves enforce server-before-worker ordering

**Deferred Decisions (Post-MVP):**

| Decision | Deferred Until |
|---|---|
| Multi-worker task queue routing | When second workflow type is added |
| Temporal namespace segmentation | If/when second environment added |
| Worker autoscaling (HPA) | After baseline load profiling |
| Archival storage for workflow history | After homelab storage review |

### Data Architecture

| Decision | Choice | Rationale |
|---|---|---|
| DB connection | `10.0.0.56:5432` (Patroni primary) | Stable bare metal IP from domain architecture |
| DB name (persistence) | `temporal` | Temporal convention |
| DB name (visibility) | `temporal_visibility` | Temporal convention; separate DB recommended |
| DB user | `temporal` | Dedicated user; no shared credentials |
| Schema migration | Helm `admintools` Job (in-band) | Idempotent, GitOps-native, auto-runs on upgrade |
| Visibility store | PostgreSQL (not Elasticsearch) | Eliminates Elasticsearch dependency entirely |
| Vault path — DB password | `secret/terminus/default/temporal/db-password` | Follows established convention |
| Vault path — visibility DB password | `secret/terminus/default/temporal/visibility-db-password` | Separate secret per DB user |
| Caching | None | Temporal handles internally |

### Authentication & Security

| Decision | Choice | Rationale |
|---|---|---|
| Temporal frontend auth | Disabled | Homelab, single user, not internet-exposed |
| Temporal Web UI auth | Disabled | Behind Traefik on private network |
| Ingress hostname | `temporal-ui.trantor.internal` | Internal DNS, consistent pattern |
| TLS | cert-manager Vault PKI issuer | Consistent with all other k3s services |
| k8s namespace | `temporal` | Clean isolation, single env |
| Worker identity | Kubernetes ServiceAccount + RBAC | No credentials in Pod spec |
| Secrets delivery | ESO `ExternalSecret` → k8s `Secret` in `temporal` ns | No secrets in git |

### API & Communication

| Decision | Choice | Rationale |
|---|---|---|
| Temporal SDK | `@temporalio/sdk` (official TypeScript) | Locked: TypeScript stack |
| Worker-to-server | gRPC (Temporal default, port 7233) | Standard — no customization needed |
| Worker connection string | `temporal-frontend.temporal.svc.cluster.local:7233` | In-cluster DNS |
| Web UI port | 8080 internal, exposed via Traefik 80/443 | Standard chart default |
| Task queue name | `terminus-platform` | Platform-scoped; all platform workflows use one queue initially |

### Worker / Application Architecture

| Decision | Choice | Rationale |
|---|---|---|
| Worker deployment | k8s `Deployment`, `replicas: 1` | Single replica sufficient; Temporal handles concurrency |
| Container registry | `ghcr.io/electricm0nk/terminus-platform-worker` | Consistent with existing repos |
| Worker Helm chart | Custom chart in `terminus.platform/helm/` | Not in Temporal server chart scope |
| Resource requests | `cpu: 100m, memory: 256Mi` | Conservative homelab sizing |
| Resource limits | `cpu: 500m, memory: 512Mi` | Adjustable after profiling |
| Health probe | `exec` against Temporal connection | SDK exposes no HTTP health endpoint |
| Logging | `pino` structured JSON to stdout | Locked in project context |

### Infrastructure & Deployment

| Decision | Choice | Rationale |
|---|---|---|
| Deployment method | ArgoCD App-of-Apps from `terminus.platform` | Consistent with all k3s workloads |
| Sync strategy | `automated: {prune: true, selfHeal: true}` | Consistent with Crossplane pattern |
| Temporal server sync wave | `"2"` | After ESO secrets (wave 1), before worker (wave 3) |
| Worker sync wave | `"3"` | Must start after Temporal server is healthy |
| Prometheus metrics | Temporal server port 9090 | Consumed by future Prometheus/Grafana initiative |

---

## Helm Chart Foundation (Step 3)

| Item | Decision |
|---|---|
| Chart | `temporalio/temporal` — official Temporal Technologies chart |
| Chart version | `1.0.0-rc.3` (app version `1.30.2`, updated 2026-03-26) |
| Chart repo | `https://go.temporal.io/helm-charts` |
| Bundled Cassandra | **Disabled** — use external PostgreSQL |
| Bundled Elasticsearch | **Disabled** — use PostgreSQL visibility store |
| Schema migrations | Keep bundled `admintools` Job — runs against PostgreSQL on install |
| Worker | **Not in chart scope** — separate TypeScript Deployment in `terminus.platform` |

---

## Implementation Patterns & Consistency Rules (Step 5)

**9 conflict areas identified** where AI agents could make different choices without explicit rules.

### Naming Patterns

| Area | Rule | Example |
|---|---|---|
| TypeScript files | `camelCase` filenames, `PascalCase` classes | `moduleRunner.ts`, `class ModuleRunner` |
| Workflow names | `PascalCase` | `DailyBriefingWorkflow` |
| Activity names | `camelCase` | `fetchWeatherActivity` |
| Task queue | Always `terminus-platform` — never hardcode alternatives | — |
| k8s resource names | `kebab-case` | `temporal-worker-deployment` |
| Vault paths | `secret/terminus/default/temporal/...` | `secret/terminus/default/temporal/db-password` |
| ESO `ExternalSecret` names | `temporal-{purpose}` | `temporal-db-credentials` |

### Structure Patterns

| Area | Rule |
|---|---|
| Tests | Co-located `*.test.ts` alongside source — never a separate `__tests__/` dir |
| Worker entrypoint | `services/temporal/src/worker.ts` — registers all workflows + activities |
| Workflows | `services/temporal/src/workflows/` — one file per workflow |
| Activities | `services/temporal/src/activities/` — one file per activity group |
| Shared types | `shared/types/src/` — all `ModulePayload`, `RunContext`, shared interfaces |
| Worker Helm chart | `helm/temporal-worker/` in repo root |
| k8s raw manifests | `k8s/` — ESO and RBAC only; everything else in Helm |
| Future services | `services/{name}/` — TypeScript services get full structure; config-only services get `helm/` + `k8s/` only |

### Format Patterns

| Area | Rule | Example |
|---|---|---|
| Log fields (required) | `timestamp`, `level`, `service`, `module`, `message`, `context` | `pino` default + `module` field |
| `service` log field | Always `terminus-platform-worker` | Never omit |
| Dates in payloads | ISO 8601 strings | `"2026-03-30T19:00:00Z"` |
| `ModulePayload` | Must declare `schema` — no untyped payloads | Per project-context interface |
| Activity terminal errors | Re-throw as `ApplicationFailure.nonRetryable(...)` | Never swallow or re-throw generic `Error` |

### Process Patterns

| Area | Rule |
|---|---|
| Activity retries | Default Temporal retry policy unless explicit override |
| Workflow determinism | **Never** call `Date.now()`, `Math.random()`, or non-deterministic APIs inside workflow functions |
| Test mocking | All external API calls mocked — no live network in test suite |
| Coverage gate | 80% line coverage; 100% on `ModulePayload` schema validation |
| Secrets | Never in source, env vars, or logs — only from k8s `Secret` via ESO |

### Mandatory Rules — All Agents MUST

- All existing and new tests pass 100% before story is ready for review
- Tests scaffolded before implementation (TDD) — no exceptions
- `ModulePayload.schema` declared on every module
- Workflow functions are pure and deterministic — all I/O in activities
- `pino` logger on every service — no `console.log` in production code

### Anti-Patterns (Forbidden)

- `Date.now()` or `Math.random()` inside a workflow function
- `console.log` in production code
- Hardcoded Vault paths or connection strings
- `maximumAttempts: 1` on activity retry policy without documented justification
- Raw k8s manifests for anything except ESO `ExternalSecret` and RBAC resources

---

## Project Structure & Boundaries (Step 6)

### Complete Project Directory Structure

`terminus.platform` is a **monorepo** hosting all platform-layer services. Each service gets its own `services/{name}/` directory. Grafana and Prometheus are platform services (serve tenant workloads, not infra-only) and will live here alongside Temporal.

```
terminus.platform/                        # target repo: github.com/electricm0nk/terminus-platform
├── README.md
├── package.json                          # npm workspaces root: ["services/*", "shared/*"]
├── tsconfig.json                         # TypeScript strict mode base config (extended per service)
├── .gitignore
├── .github/
│   └── workflows/
│       ├── ci.yml                        # runs tests for all changed services on PR
│       └── publish-images.yml            # builds + pushes worker images to GHCR on merge
│
├── services/
│   │
│   ├── temporal/                         # Temporal worker (TypeScript) — THIS INITIATIVE
│   │   ├── package.json                  # workspace package; depends on shared/types
│   │   ├── tsconfig.json                 # extends root tsconfig
│   │   ├── vitest.config.ts              # coverage thresholds: 80% line, 100% schema
│   │   └── src/
│   │       ├── worker.ts                 # ENTRYPOINT: registers all workflows + activities
│   │       ├── workflows/
│   │       │   ├── dailyBriefingWorkflow.ts
│   │       │   └── dailyBriefingWorkflow.test.ts   # co-located test
│   │       ├── activities/
│   │       │   ├── briefingActivities.ts            # one file per activity group
│   │       │   └── briefingActivities.test.ts
│   │       └── lib/
│   │           ├── logger.ts             # pino logger factory (service field pre-set)
│   │           └── errors.ts             # ApplicationFailure helpers
│   │
│   ├── grafana/                          # (future: terminus-platform-grafana initiative)
│   │   ├── helm/                         # Grafana Helm values overrides
│   │   └── k8s/                          # ESO ExternalSecrets, ConfigMaps
│   │
│   └── prometheus/                       # (future: terminus-platform-prometheus initiative)
│       ├── helm/                         # Prometheus Helm values overrides
│       └── k8s/                          # ESO ExternalSecrets, alert rules
│
├── shared/
│   ├── types/                            # ModulePayload, RunContext, Module interface
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── module.ts                 # Module interface, ModulePayload, RunContext
│   │       ├── module.test.ts            # schema validation — 100% coverage required
│   │       └── index.ts                  # barrel export
│   └── lib/                              # shared logger factory, error helpers
│       ├── package.json
│       └── src/
│           ├── logger.ts
│           └── errors.ts
│
├── helm/
│   └── temporal-worker/                  # Helm chart for Temporal worker Deployment
│       ├── Chart.yaml
│       ├── values.yaml                   # defaults (image, resources, taskQueue)
│       └── templates/
│           ├── deployment.yaml
│           ├── serviceaccount.yaml
│           └── rbac.yaml
│
└── k8s/
    ├── namespace.yaml                    # temporal namespace
    ├── external-secret-db.yaml           # ESO ExternalSecret: temporal-db-credentials
    ├── external-secret-visibility-db.yaml # ESO ExternalSecret: temporal-visibility-db-credentials
    └── rbac-worker.yaml                  # ClusterRole + binding for worker ServiceAccount
```

**Monorepo structural rules (all future platform services must follow):**
- TypeScript services: full `services/{name}/` with `package.json`, `tsconfig.json`, `src/`, co-located tests
- Non-TypeScript services (Grafana, Prometheus): `services/{name}/helm/` and `services/{name}/k8s/` only
- All TypeScript services extend root `tsconfig.json` and declare workspace dependency on `shared/types`
- `shared/types` and `shared/lib` are the only cross-service shared code — no service imports from another service

### Architectural Boundaries

**Temporal Server boundary** (managed by `temporalio/temporal` Helm chart — not owned by this repo):
- `temporal-frontend` service: gRPC port 7233 — worker connects here
- `temporal-web` service: HTTP port 8080 — UI via Traefik ingress
- `temporal-admintools` Job: runs schema migrations on install/upgrade

**Worker boundary** (owned by `terminus.platform`):
- Connects outbound to `temporal-frontend.temporal.svc.cluster.local:7233`
- No inbound connections — pull model (worker polls Temporal for tasks)
- Reads k8s `Secret` for any runtime credentials (via ESO, mounted as env vars)

**External boundaries:**
- PostgreSQL at `10.0.0.56:5432` — Temporal server connects directly (not the worker)
- Vault at `vault.trantor.internal` — ESO syncs secrets; worker never calls Vault directly
- ArgoCD watches `terminus.platform` git repo for Helm chart changes

### Integration Points

**Data flow — workflow execution:**
```
External trigger (cron / event)
  → Temporal server schedules workflow
  → Worker polls task queue `terminus-platform`
  → Worker executes DailyBriefingWorkflow
  → Workflow calls activities (all I/O here)
  → Activities return ModulePayload
  → Workflow assembles BriefingContext
  → Render activity applies persona pass
  → Delivery activity sends output (email/SMS)
  → Workflow completes, result stored in PostgreSQL visibility store
```

**Secret delivery flow:**
```
Vault KV v2 (secret/terminus/default/temporal/...)
  → ESO ExternalSecret polls on schedule
  → k8s Secret created/updated in `temporal` namespace
  → Temporal Helm chart references Secret for DB credentials
  → Worker Deployment references Secret for any runtime credentials
```

### Requirements to Structure Mapping

| Concern | Location |
|---|---|
| Temporal server install | `temporalio/temporal` Helm chart (external) |
| DB provisioning story | Patroni admin task — separate story, not in this repo |
| Vault secret paths | `k8s/external-secret-*.yaml` |
| Worker application code | `src/workflows/`, `src/activities/`, `src/types/` |
| Worker k8s deployment | `helm/temporal-worker/` |
| CI/CD | `.github/workflows/` |
| All unit tests | Co-located `*.test.ts` alongside source files |

---

## Architecture Validation Results (Step 7)

### Coherence Validation ✅

**Decision Compatibility:**
All technology choices are compatible. The `temporalio/temporal` Helm chart's external PostgreSQL support (`sql.driver: postgres`, Cassandra/Elasticsearch disabled) aligns exactly with the data architecture decisions. ESO `ExternalSecret` resources reference the Vault paths specified in the security decisions — no mismatches. ArgoCD sync waves (ESO=1, Temporal server=2, Worker=3) enforce the correct dependency chain: secrets exist before server starts, server is healthy before worker attempts to connect on port 7233. The TypeScript monorepo structure with npm workspaces is fully compatible with the `@temporalio/sdk` official package and `vitest` test runner.

*Note:* Step 2 context refers to `@temporalio/worker` (legacy package name); Step 4 correctly specifies `@temporalio/sdk` (the current official unified package). Step 4 is authoritative — no conflicting decision.

**Pattern Consistency:**
Naming conventions span all layers without collision: `kebab-case` for k8s resources, `camelCase` for TypeScript activity files and function names, `PascalCase` for workflow class names, `secret/terminus/default/temporal/...` for Vault paths, `temporal-{purpose}` for ESO names. Implementation patterns (determinism rules, `ApplicationFailure`, pino logging) are internally consistent and executable by AI agents without ambiguity.

**Structure Alignment:**
The monorepo structure directly enables the architectural decisions. `helm/temporal-worker/` is completely distinct from the Temporal server Helm chart — no co-tenancy confusion. The `k8s/` directory scope (ESO + RBAC only) is consistent with the anti-pattern rule forbidding raw manifests for application resources. The `shared/` split correctly isolates type definitions from implementation helpers, enabling safe cross-service type sharing without coupling implementations.

---

### Requirements Coverage Validation ✅

**Initiative & Track Coverage (tech-change — no formal PRD):**

| Locked Requirement | Architectural Coverage |
|---|---|
| TypeScript strict mode | Root `tsconfig.json` strict base; all `services/` extend it |
| Node.js LTS runtime | Worker Deployment image from GHCR |
| Helm required for all k8s workloads | `temporalio/temporal` chart + custom `helm/temporal-worker/` |
| Temporal for workflow orchestration | Full server deployment + TypeScript worker |
| k3s on Proxmox + ArgoCD GitOps | App-of-Apps pattern, sync waves, `prune: true`, `selfHeal: true` |
| `pino` logging required | Step 5 mandatory rule; `service: terminus-platform-worker` always set |
| `vitest` for testing | `vitest.config.ts` with coverage thresholds (80% line, 100% schema) |
| Vault + ESO for secrets | ESO `ExternalSecret` resources; no secrets in git; never in logs |
| DB provisioning in-scope | Captured as prerequisite Story Zero — own story, own PR |
| Multi-service platform monorepo | `services/`, `shared/`, npm workspaces, accommodates Grafana + Prometheus |

**Non-Functional Requirements:**

- **Security:** Vault/ESO secret chain, dedicated `temporal` k8s namespace, ServiceAccount + RBAC for worker, no credentials in source ✅
- **Observability:** Temporal server exposes metrics on port 9090; consumed by future Prometheus initiative; structured logging via pino ✅
- **Reliability:** Sync waves enforce startup ordering; `selfHeal: true` corrects drift; Temporal's built-in retry policies govern activity reliability ✅
- **Scalability:** `replicas: 1` baseline with explicitly deferred HPA pending load profiling; Patroni provides DB HA ✅

---

### Implementation Readiness Validation ✅

**Decision Completeness:**
All critical decisions carry specific, unambiguous values — no "TBD" entries in blocking cells:
- Chart version: `1.0.0-rc.3` (app `1.30.2`) — pinned
- DB connection: `10.0.0.56:5432` — stable Patroni primary IP
- Task queue: `terminus-platform` — named constant
- Registry: `ghcr.io/electricm0nk/terminus-platform-worker` — fully qualified
- Vault paths: `secret/terminus/default/temporal/db-password` and `...(visibility-db-password` — fully qualified
- k8s namespace: `temporal` — confirmed single-env

**Structure Completeness:**
Full directory tree is defined to filename level — including CI workflow files, Helm template names, and k8s manifest names. An implementation agent can scaffold the repo without making any unspecified architectural decisions.

**Pattern Completeness:**
9 conflict areas addressed with concrete rules and examples. Anti-patterns are enumerated explicitly. TDD mandate is unambiguous with per-file-type coverage thresholds. Process rules (determinism enforcement, `ApplicationFailure.nonRetryable()`) are specific enough to be implemented without interpretation.

---

### Gap Analysis Results

**Critical Gaps (prerequisite stories — must land before Temporal implementation):**

1. **Story Zero A — Monorepo scaffold:** `terminus.platform` is currently a stub repo. No `package.json`, `tsconfig.json`, `shared/` workspaces, or `.github/workflows/` exist. This story initializes the full monorepo structure (root workspace config, `shared/types`, `shared/lib`, base CI skeleton). All subsequent implementation stories depend on this landing first.

2. **Story Zero B — DB provisioning:** Patroni must have the `temporal` database, dedicated `temporal` DB user, and `temporal_visibility` database created before the Temporal Helm chart can run schema migrations. This is a hard prerequisite for the Temporal server story.

**Important Gaps (don't block implementation; resolve at story time):**

1. **ArgoCD Application manifest location:** Not specified in this document. Per established terminus pattern (consistent with crossplane initiatives), the `Application` CR for `terminus.platform` lives in the terminus infrastructure/governance repo — not in `terminus.platform` itself. Implementation agent should follow the existing App-of-Apps pattern.

2. **ESO ClusterSecretStore name:** Not specified. Implementation agent writing `ExternalSecret` manifests must resolve this from domain architecture at story time. Expected to be the existing ClusterSecretStore already serving other terminus workloads.

**Nice-to-Have Gaps (no action required):**

1. Helm values file path structure for PostgreSQL overrides — discoverable from chart documentation during implementation
2. GHCR image build CI trigger specifics (branch filter, tag pattern) — reasonable to define in the CI story

---

### Validation Issues Addressed

**No critical blocking issues found.** The two Story Zero gaps are known, intentional in-scope items already identified during architectural review — not defects.

The `@temporalio/worker` / `@temporalio/sdk` language inconsistency in Step 2 is a description artifact from context analysis, not an architectural conflict. Step 4 decision (`@temporalio/sdk`) is authoritative.

ArgoCD Application manifest location follows established terminus convention and does not require specification in this document.

---

### Architecture Completeness Checklist

**✅ Requirements Analysis**

- [x] Project context thoroughly analyzed (initiative config, project-context.md, domain architecture)
- [x] Scale and complexity assessed (homelab single-env, single k3s cluster)
- [x] Technical constraints identified (TypeScript strict, Helm required, Patroni, Vault)
- [x] Cross-cutting concerns mapped (secrets, logging, GitOps, sync waves)

**✅ Architectural Decisions**

- [x] Critical decisions documented with versions (chart `1.0.0-rc.3`, app `1.30.2`)
- [x] Technology stack fully specified (TypeScript, `@temporalio/sdk`, vitest, pino, PostgreSQL)
- [x] Integration patterns defined (ESO, ArgoCD App-of-Apps, sync waves)
- [x] Performance considerations addressed (resource requests/limits, replica count, deferral documented)

**✅ Implementation Patterns**

- [x] Naming conventions established (all layers: k8s, TypeScript, Vault, ESO)
- [x] Structure patterns defined (co-located tests, one file per workflow/activity group)
- [x] Communication patterns specified (gRPC port 7233, in-cluster DNS, task queue name)
- [x] Process patterns documented (determinism, ApplicationFailure, TDD, coverage gates)

**✅ Project Structure**

- [x] Complete directory structure defined (to filename level)
- [x] Component boundaries established (worker boundary, server boundary, external boundaries)
- [x] Integration points mapped (data flow, secret delivery flow)
- [x] Requirements to structure mapping complete

---

### Architecture Readiness Assessment

**Overall Status:** READY FOR IMPLEMENTATION

**Confidence Level:** High — all critical decisions are specific and unambiguous; patterns are comprehensive; structure is fully defined; the only open items are operational details (ArgoCD App location, ESO store name) that follow established terminus conventions and are resolvable at story time.

**Key Strengths:**
- Monorepo design future-proofs the platform layer for Grafana + Prometheus without rework
- Sync wave ordering eliminates a whole class of race-condition bugs at deployment time
- Explicit determinism rules + ApplicationFailure patterns give AI agents clear guardrails
- PostgreSQL visibility store eliminates Elasticsearch operational burden entirely
- Two prerequisite stories cleanly identified and bounded — no hidden blockers

**Areas for Future Enhancement:**
- Worker autoscaling (HPA) after baseline load profiling
- Temporal namespace segmentation if second environment is added
- Archival storage for long-running workflow history
- Multi-worker task queue routing when workflow types diverge

---

### Implementation Handoff

**AI Agent Guidelines:**

- Follow all architectural decisions exactly as documented — no improvisation on technology choices
- Use implementation patterns consistently: naming, structure, logging, and error handling rules are non-negotiable
- Respect monorepo boundaries: `shared/types` and `shared/lib` only; no cross-service imports
- Refer to this document for all architectural questions before introducing new patterns

**Story Execution Order:**
1. **Story Zero A:** Initialize monorepo scaffold (`terminus.platform` root structure, npm workspaces, `shared/types`, `shared/lib`, base CI)
2. **Story Zero B:** Patroni DB + user provisioning (`temporal` and `temporal_visibility` databases, dedicated user, Vault secrets)
3. **Temporal server story:** Helm values, ESO ExternalSecrets, k8s namespace, ArgoCD Application
4. **Temporal worker story:** TypeScript worker scaffold, `services/temporal/src/`, Helm chart, CI image build

---

## Deployment Pipeline & Usage Patterns

This section documents the **operational interface** for the Temporal platform post-implementation — which changes to which repository trigger which downstream tool.

### Source of Truth Mapping

Changes to `terminus.platform` land on `develop` as the default integration path. Promote to `main` only for the production release path.

| Changed path in `terminus.platform` | Tool | What happens |
|---|---|---|
| `k8s/**` | ArgoCD `temporal-infra` (auto-sync) | Namespace, RBAC, ESO ExternalSecrets, cert-manager Certificate, and Ingress applied to `temporal` namespace |
| `helm/temporal-server/values.yaml` | ArgoCD `temporal-server` (auto-sync, wave 2) | Temporal server Helm upgrade using updated values |
| `helm/temporal-worker/**` | ArgoCD `temporal-worker` (auto-sync, wave 3) | Worker Deployment Helm upgrade |
| `services/temporal/**` or `shared/**` | GitHub Actions CI → GHCR | New image built and pushed as `:sha-<hash>` + `:latest`; running pod picks up on next restart (`imagePullPolicy: Always`) |

### ArgoCD Application Ownership

All three ArgoCD Applications live in `terminus.infra/platforms/k3s/argocd/apps/`:

| Application | Sync wave | Source path in `terminus.platform` |
|---|---|---|
| `temporal-infra` | (no wave — runs first) | `k8s/` |
| `temporal-server` | `"2"` | Helm chart from `go.temporal.io` + `helm/temporal-server/values.yaml` |
| `temporal-worker` | `"3"` | `helm/temporal-worker/` |

Changing the Temporal Helm chart version (`temporalio/temporal`) requires editing `temporal-server.yaml` in `terminus.infra`, not `terminus.platform`.

### OpenTofu Scope

OpenTofu (`terminus.infra/tofu/`) manages the **infrastructure prereq layer only** — VMs, k3s cluster, Patroni, and Vault. There is no `tofu/environments/temporal`; Temporal workloads are managed entirely by ArgoCD + Helm.

OpenTofu is a **one-time prerequisite** — `tofu apply` must have provisioned k3s and Patroni before Temporal can be deployed. It does not participate in the day-to-day Temporal deployment pipeline.

### One-Time Prerequisites (Manual, Per Environment)

These steps are completed once per environment and do not need to be repeated for normal application changes:

| Step | Command / Tool | Repository |
|---|---|---|
| k3s cluster | `tofu apply` in `tofu/environments/k3s/dev` | `terminus.infra` |
| Patroni infra | `tofu apply` in `tofu/environments/postgres/dev` | `terminus.infra` |
| DB provisioning | `bash docs/scripts/run-db-provision.sh` | `terminus.platform` |
| DB credentials | `vault kv put secret/terminus/default/temporal/db-password value=...` | Vault |
| GHCR pull token | `vault kv put secret/terminus/default/ghcr/pull-token value=...` | Vault |
