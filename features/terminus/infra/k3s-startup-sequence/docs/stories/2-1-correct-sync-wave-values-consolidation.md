# Story 2.1: Correct sync-wave values for actions-runner, workloads, ingress, and servicemonitors

## Status: ready-for-dev

## Story

As a platform engineer,
I want the existing `argocd.argoproj.io/sync-wave` annotation values corrected on 23 Application objects that currently carry incorrect wave numbers,
So that actions-runner loads after temporal-workers, all workload apps load after infra namespace-prep, all ingress apps load after their backing services, and monitoring servicemonitors are last.

## Acceptance Criteria

- **Given** `actions-runner.yaml` has `sync-wave: "2"` (deploying alongside temporal-server)
- **When** the value is changed to `"4"`
- **Then** actions-runner deploys after temporal-workers (wave 3) are healthy

- **Given** the following 6 fourdogs workload apps have `sync-wave: "1"`:
  `fourdogs-central.yaml`, `fourdogs-central-dev.yaml`, `fourdogs-emailfetcher.yaml`,
  `fourdogs-emailfetcher-dev.yaml`, `fourdogs-kaylee-agent.yaml`, `fourdogs-kaylee-agent-dev.yaml`
- **When** all values are changed to `"6"`
- **Then** all fourdogs workload apps deploy after namespace-prep (wave 1) is complete and temporal is ready (wave 2–3)

- **Given** the following 11 apps have `sync-wave: "2"`:
  `grafana.yaml`, `grafana-dev.yaml`, `inference-gateway.yaml`, `inference-gateway-dev.yaml`,
  `influxdb.yaml`, `influxdb-dev.yaml`, `prometheus.yaml`, `prometheus-dev.yaml`,
  `ollama.yaml`, `terminus-portal.yaml`, `terminus-portal-dev.yaml`
- **When** all values are changed to `"6"`
- **Then** all these workload apps deploy at wave 6, after infra and temporal tiers

- **Given** the following 8 ingress apps have incorrect wave values:
  `grafana-ingress.yaml` (`"3"`→`"7"`), `grafana-dev-ingress.yaml` (`"3"`→`"7"`),
  `inference-gateway-ingress.yaml` (`"3"`→`"7"`), `inference-gateway-dev-ingress.yaml` (`"3"`→`"7"`),
  `influxdb-ingress.yaml` (`"3"`→`"7"`), `influxdb-dev-ingress.yaml` (`"3"`→`"7"`),
  `prometheus-ingress.yaml` (`"3"`→`"7"`), `terminus-portal-dev-ingress.yaml` (`"3"`→`"7"`)
- **When** all values are changed to `"7"`
- **Then** all ingress apps deploy after their backing services are scheduled (wave 6)

- **Given** `temporal-worker-ingress.yaml` and `temporal-worker-dev-ingress.yaml` have `sync-wave: "4"`
- **When** both values are changed to `"7"`
- **Then** temporal-worker ingress deploys after workload tier, consistent with all other ingress apps

- **Given** `monitoring-servicemonitors.yaml` has `sync-wave: "3"`
- **When** the value is changed to `"8"`
- **Then** servicemonitors are applied last, after all scrape targets (wave 6 workloads) are scheduled

- **Given** all 23 changes are applied and ArgoCD syncs `terminus-infra-k3s-root`
- **When** the following command is run:
  ```bash
  kubectl get applications -n argocd \
    -o custom-columns=\
  'NAME:.metadata.name,WAVE:.metadata.annotations.argocd\.argoproj\.io/sync-wave' \
    --sort-by='.metadata.annotations.argocd\.argoproj\.io/sync-wave'
  ```
- **Then** the output shows:
  - Wave 4: `actions-runner`
  - Wave 6: `fourdogs-central`, `fourdogs-central-dev`, `fourdogs-emailfetcher`, `fourdogs-emailfetcher-dev`, `fourdogs-kaylee-agent`, `fourdogs-kaylee-agent-dev`, `grafana`, `grafana-dev`, `inference-gateway`, `inference-gateway-dev`, `influxdb`, `influxdb-dev`, `prometheus`, `prometheus-dev`, `ollama`, `terminus-portal`, `terminus-portal-dev`
  - Wave 7: `grafana-ingress`, `grafana-dev-ingress`, `inference-gateway-ingress`, `inference-gateway-dev-ingress`, `influxdb-ingress`, `influxdb-dev-ingress`, `prometheus-ingress`, `terminus-portal-dev-ingress`, `temporal-worker-ingress`, `temporal-worker-dev-ingress`
  - Wave 8: `monitoring-servicemonitors`

## Tasks / Subtasks

- [ ] Task 1: Fix actions-runner wave (2→4)
  - [ ] Edit `platforms/k3s/argocd/apps/actions-runner.yaml` — change `sync-wave: "2"` to `sync-wave: "4"`
- [ ] Task 2: Fix fourdogs workload waves (1→6) — 6 files
  - [ ] `fourdogs-central.yaml`: `"1"` → `"6"`
  - [ ] `fourdogs-central-dev.yaml`: `"1"` → `"6"`
  - [ ] `fourdogs-emailfetcher.yaml`: `"1"` → `"6"`
  - [ ] `fourdogs-emailfetcher-dev.yaml`: `"1"` → `"6"`
  - [ ] `fourdogs-kaylee-agent.yaml`: `"1"` → `"6"`
  - [ ] `fourdogs-kaylee-agent-dev.yaml`: `"1"` → `"6"`
- [ ] Task 3: Fix observability and app workload waves (2→6) — 11 files
  - [ ] `grafana.yaml`: `"2"` → `"6"`
  - [ ] `grafana-dev.yaml`: `"2"` → `"6"`
  - [ ] `inference-gateway.yaml`: `"2"` → `"6"`
  - [ ] `inference-gateway-dev.yaml`: `"2"` → `"6"`
  - [ ] `influxdb.yaml`: `"2"` → `"6"`
  - [ ] `influxdb-dev.yaml`: `"2"` → `"6"`
  - [ ] `prometheus.yaml`: `"2"` → `"6"`
  - [ ] `prometheus-dev.yaml`: `"2"` → `"6"`
  - [ ] `ollama.yaml`: `"2"` → `"6"`
  - [ ] `terminus-portal.yaml`: `"2"` → `"6"`
  - [ ] `terminus-portal-dev.yaml`: `"2"` → `"6"`
- [ ] Task 4: Fix ingress waves (3→7 and 4→7) — 10 files
  - [ ] `grafana-ingress.yaml`: `"3"` → `"7"`
  - [ ] `grafana-dev-ingress.yaml`: `"3"` → `"7"`
  - [ ] `inference-gateway-ingress.yaml`: `"3"` → `"7"`
  - [ ] `inference-gateway-dev-ingress.yaml`: `"3"` → `"7"`
  - [ ] `influxdb-ingress.yaml`: `"3"` → `"7"`
  - [ ] `influxdb-dev-ingress.yaml`: `"3"` → `"7"`
  - [ ] `prometheus-ingress.yaml`: `"3"` → `"7"`
  - [ ] `terminus-portal-dev-ingress.yaml`: `"3"` → `"7"`
  - [ ] `temporal-worker-ingress.yaml`: `"4"` → `"7"`
  - [ ] `temporal-worker-dev-ingress.yaml`: `"4"` → `"7"`
- [ ] Task 5: Fix servicemonitor wave (3→8) — 1 file
  - [ ] `monitoring-servicemonitors.yaml`: `"3"` → `"8"`
- [ ] Task 6: Commit all 23 files
  - [ ] Commit message: `fix(argocd): correct sync-wave values for workloads, ingress, and servicemonitors`
- [ ] Task 7: After ArgoCD sync, run wave-compliance verification command and confirm expected output

## Dev Notes

**Change is value-only:** Every edit in this story is changing an existing string value (e.g. `"2"` → `"6"`) inside a `metadata.annotations` block. No structural YAML changes required.

**All files are in:** `platforms/k3s/argocd/apps/` — all paths are relative to repository root.

**Wave compliance command** (from acceptance criteria above) is the post-merge smoke gate. Run it after `terminus-infra-k3s-root` syncs from `main`.

**Rollback:** `git revert <commit-sha>` — annotation changes are ArgoCD-metadata-only, no cluster state is modified by the revert.

**ArgoCD sync-wave ordering contract:**
- Wave 2: `temporal-server` (already correct — do not change)
- Wave 3: `temporal-worker`, `temporal-worker-dev` (already correct — do not change)
- Wave 4: `actions-runner` (this story)
- Wave 6: all application workloads (this story)
- Wave 7: all ingress objects (this story)
- Wave 8: `monitoring-servicemonitors` (this story)

**Reference:** `docs/terminus/infra/k3s-startup-sequence/architecture.md` — complete wave map, change table, and ADRs.

## Dev Agent Record

### Agent Model Used: ``
### Completion Notes List

### Change List
- `platforms/k3s/argocd/apps/actions-runner.yaml` — wave 2→4
- `platforms/k3s/argocd/apps/fourdogs-central.yaml` — wave 1→6
- `platforms/k3s/argocd/apps/fourdogs-central-dev.yaml` — wave 1→6
- `platforms/k3s/argocd/apps/fourdogs-emailfetcher.yaml` — wave 1→6
- `platforms/k3s/argocd/apps/fourdogs-emailfetcher-dev.yaml` — wave 1→6
- `platforms/k3s/argocd/apps/fourdogs-kaylee-agent.yaml` — wave 1→6
- `platforms/k3s/argocd/apps/fourdogs-kaylee-agent-dev.yaml` — wave 1→6
- `platforms/k3s/argocd/apps/grafana.yaml` — wave 2→6
- `platforms/k3s/argocd/apps/grafana-dev.yaml` — wave 2→6
- `platforms/k3s/argocd/apps/inference-gateway.yaml` — wave 2→6
- `platforms/k3s/argocd/apps/inference-gateway-dev.yaml` — wave 2→6
- `platforms/k3s/argocd/apps/influxdb.yaml` — wave 2→6
- `platforms/k3s/argocd/apps/influxdb-dev.yaml` — wave 2→6
- `platforms/k3s/argocd/apps/prometheus.yaml` — wave 2→6
- `platforms/k3s/argocd/apps/prometheus-dev.yaml` — wave 2→6
- `platforms/k3s/argocd/apps/ollama.yaml` — wave 2→6
- `platforms/k3s/argocd/apps/terminus-portal.yaml` — wave 2→6
- `platforms/k3s/argocd/apps/terminus-portal-dev.yaml` — wave 2→6
- `platforms/k3s/argocd/apps/grafana-ingress.yaml` — wave 3→7
- `platforms/k3s/argocd/apps/grafana-dev-ingress.yaml` — wave 3→7
- `platforms/k3s/argocd/apps/inference-gateway-ingress.yaml` — wave 3→7
- `platforms/k3s/argocd/apps/inference-gateway-dev-ingress.yaml` — wave 3→7
- `platforms/k3s/argocd/apps/influxdb-ingress.yaml` — wave 3→7
- `platforms/k3s/argocd/apps/influxdb-dev-ingress.yaml` — wave 3→7
- `platforms/k3s/argocd/apps/prometheus-ingress.yaml` — wave 3→7
- `platforms/k3s/argocd/apps/terminus-portal-dev-ingress.yaml` — wave 3→7
- `platforms/k3s/argocd/apps/temporal-worker-ingress.yaml` — wave 4→7
- `platforms/k3s/argocd/apps/temporal-worker-dev-ingress.yaml` — wave 4→7
- `platforms/k3s/argocd/apps/monitoring-servicemonitors.yaml` — wave 3→8
