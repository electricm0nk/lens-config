# Story 8.1: Portal2 Deployment — Release Pipeline Wiring and Production Launch

Status: done

## Story

As a platform operator,
I want `terminus-portal` (portal2) deployed to both the dev and production k3s environments via the standard Temporal release pipeline,
So that every future portal change flows through the automated GitOps release contract (build → promote → Temporal → ArgoCD → smoke test).

## Acceptance Criteria

1. A `release.yml` GitHub Actions workflow exists in the portal repo that fires on PR merge to `develop` (dev release) or `main` (prod release), promotes the image tag to `terminus.infra`, and starts a Temporal `ReleaseWorkflow`
2. `terminus.infra` has a dev environment wired: `values-dev.yaml`, three ArgoCD Application manifests (`terminus-portal-dev-infra`, `terminus-portal-dev`, `terminus-portal-dev-ingress`), and a dev ingress manifest pointing at `portal-dev.trantor.internal`
3. The prod ArgoCD application (`terminus-portal`) has `syncPolicy.automated` removed — Temporal controls rollout start
4. Ansible verify playbooks exist for dev and prod: `verify-terminus-portal-dev-deploy.yml`, `verify-terminus-portal-deploy.yml` — each asserts the portal HTML is returned from the relevant hostname via HTTP
5. Semaphore templates `verify-terminus-portal-dev-deploy` and `verify-terminus-portal-deploy` are declared in `terminus.infra/tofu/environments/semaphore/` and applied to project 3 (`terminus-infra`)
6. CoreDNS coredns-custom configmap and `/etc/hosts` on `vault.trantor.internal` are updated for `portal-dev.trantor.internal → 10.0.0.126`
7. Dev release: `feature/portal2` merged to `develop` via PR → `release.yml` fires → Temporal workflow `release-terminus-portal-dev-{sha}` completes all activities → ArgoCD app `terminus-portal-dev` reaches `Synced/Healthy` → smoke test passes → portal2 UI visible at `https://portal-dev.trantor.internal`
8. Production release: `develop` merged to `main` via PR → `release.yml` fires → Temporal workflow completes → ArgoCD app `terminus-portal` reaches `Synced/Healthy` → smoke test passes → portal2 UI visible at `https://portal.trantor.internal`

## Tasks / Subtasks

- [x] Task 1: Add `develop` branch and `release.yml` to the portal service repo (AC: #1)
  - [x] Create `develop` branch from `feature/portal2` (contains all 18 completed stories)
  - [x] Create `.github/workflows/release.yml` with two jobs:
    - `build-and-push` (cloud runner): build Docker image, push to `ghcr.io/electricm0nk/terminus-portal:{sha}`
    - `promote-and-release` (self-hosted `trantor-internal` runner):
      - Clone `terminus.infra`, checkout the correct branch (`develop` for dev, `main` for prod)
      - `sed` the image tag in the correct values file (`values-dev.yaml` or `values.yaml`)
      - Commit and push to `terminus.infra`
      - Start Temporal `ReleaseWorkflow` via `kubectl exec` into `temporal-admintools` pod
      - Temporal input: `{"ServiceName":"terminus-portal","Environment":"<env>","ProvisionDB":false,"ImageTag":"<sha>"}`
  - [x] CI job (`ci.yml`) must run `npm test` only (Vitest unit tests) — NOT `test:e2e` (Playwright requires a live server; not appropriate for CI gate)
  - [x] Verify `ci.yml` `npm test` script runs `vitest run` (not `playwright test`)

- [x] Task 2: Create dev environment in `terminus.infra` (AC: #2)
  - [ ] `platforms/k3s/helm/terminus-portal/values-dev.yaml`:
    ```yaml
    image:
      tag: latest  # overwritten by release.yml on each merge to develop
    ingress:
      host: portal-dev.trantor.internal
    ```
  - [ ] `platforms/k3s/argocd/apps/terminus-portal-dev-infra.yaml` — ArgoCD Application syncing `deploy/k8s/` (namespace + GHCR ExternalSecret) from `develop` branch, targetNamespace `terminus-portal-dev`
  - [ ] `platforms/k3s/argocd/apps/terminus-portal-dev.yaml` — ArgoCD Application syncing Helm chart from `develop` branch, valueFiles: `[values.yaml, values-dev.yaml]`, targetNamespace `terminus-portal-dev`, NO `syncPolicy.automated` (Temporal-gated)
  - [ ] `platforms/k3s/argocd/apps/terminus-portal-dev-ingress.yaml` — ArgoCD Application syncing dev ingress from `develop` branch, targetNamespace `terminus-portal-dev`
  - [ ] `platforms/k3s/k8s/terminus-portal-dev-ingress/ingress.yaml` — Traefik Ingress for `portal-dev.trantor.internal`, namespace `terminus-portal-dev`, service `terminus-portal`, port 80
  - [ ] Namespace manifest for `terminus-portal-dev` (or add to existing `deploy/k8s/namespace.yaml` with a separate dev namespace manifest in `terminus.infra`)
  - [ ] GHCR ExternalSecret for `terminus-portal-dev` namespace (same as prod pattern but namespace override)
  - [ ] Commit all to `terminus.infra` `develop` branch

- [x] Task 3: Update prod ArgoCD app in `terminus.infra` to remove auto-sync (AC: #3)
  - [ ] Edit `platforms/k3s/argocd/apps/terminus-portal.yaml` — remove `syncPolicy.automated` block if present
  - [ ] Commit to `terminus.infra` `main` branch
  - [ ] Re-sync root app: `argocd app sync terminus-infra-k3s-root` to pick up the change

- [x] Task 4: Create Ansible verify playbooks in `terminus.infra` (AC: #4)
  - [ ] `platforms/k3s/ansible/playbooks/verify-terminus-portal-dev-deploy.yml`:
    ```yaml
    - name: Verify terminus-portal dev deployment
      hosts: localhost
      tasks:
        - name: Check portal-dev returns HTTP 200
          uri:
            url: "http://portal-dev.trantor.internal"
            method: GET
            validate_certs: false
            status_code: 200
          register: result
        - name: Assert HTML response body contains portal content
          assert:
            that: "'<!doctype html>' in result.content | lower or '<html' in result.content | lower"
    ```
  - [ ] `platforms/k3s/ansible/playbooks/verify-terminus-portal-deploy.yml` — same pattern for `http://portal.trantor.internal`
  - [ ] Verify: playbooks use external `*.trantor.internal` hostnames (NOT cluster-internal `svc.cluster.local`)
  - [ ] Verify: `validate_certs: false` (self-signed TLS on `.trantor.internal`)
  - [ ] Commit to `terminus.infra`

- [x] Task 5: Declare Semaphore templates in `terminus.infra` OpenTofu (AC: #5)
  - [ ] Edit `terminus.infra/tofu/environments/semaphore/templates.tf` (or equivalent) to add:
    - Template `verify-terminus-portal-dev-deploy` → playbook `verify-terminus-portal-dev-deploy.yml`, project 3
    - Template `verify-terminus-portal-deploy` → playbook `verify-terminus-portal-deploy.yml`, project 3
  - [ ] Note: NO `seed-terminus-portal-secrets` template needed — portal has no app secrets (only GHCR pull secret, already global in `secret/terminus/default/ghcr/pull-token`)
  - [ ] Note: NO `db-provision-terminus-portal` template needed — portal is a static SPA, no database
  - [ ] Run OpenTofu apply via the `terminus.infra` Semaphore reconcile workflow (merge to `main` triggers it)
  - [ ] Verify templates appear in Semaphore UI project 3 (`terminus-infra`): `https://semaphore.trantor.internal`

- [x] Task 6: Update DNS for dev hostname (AC: #6)
  - [ ] Update CoreDNS coredns-custom configmap to resolve `portal-dev.trantor.internal`:
    ```bash
    ssh vault.trantor.internal "KUBECONFIG=~/.kube/config kubectl get configmap -n kube-system coredns-custom -o yaml"
    # Add: 10.0.0.126 portal-dev.trantor.internal
    kubectl patch configmap -n kube-system coredns-custom --type=merge \
      -p '{"data":{"trantor.server":"...existing + portal-dev.trantor.internal"}}'
    ```
  - [ ] Add `/etc/hosts` entry on `vault.trantor.internal`:
    ```bash
    ssh vault.trantor.internal "echo '10.0.0.126 portal-dev.trantor.internal' | sudo tee -a /etc/hosts"
    ```
  - [ ] Verify resolution from vault VM:
    ```bash
    ssh vault.trantor.internal "nslookup portal-dev.trantor.internal"
    ```

- [x] Task 7: Dev release — merge `feature/portal2` to `develop` and verify (AC: #7)
  - [ ] Create PR: `feature/portal2` → `develop` in the portal repo
  - [ ] Verify `ci.yml` passes (unit tests green) before merging
  - [ ] Merge PR → `release.yml` fires
  - [ ] Monitor release progress:
    - **GitHub Actions:** `promote-and-release` job on self-hosted `trantor-internal` runner
    - **Temporal UI:** `https://temporal-ui.trantor.internal` — workflow `release-terminus-portal-dev-{sha[:8]}` in `default` namespace — expected activity sequence: `WaitForArgoCDSync` → `RunSmokeTest`
      - Note: `SeedSecrets` succeeds trivially if template returns OK for services with no secrets (verify this behavior); alternatively, if the `ReleaseWorkflow` requires `SeedSecrets` to have a real template, it MUST exist in Semaphore project 3 — create a no-op `seed-terminus-portal-secrets.yml` playbook as a stub
    - **Worker logs:**
      ```bash
      ssh vault.trantor.internal "KUBECONFIG=~/.kube/config kubectl logs -n temporal deployment/temporal-worker --since=5m 2>&1 | grep -v TMPRL | tail -30"
      ```
    - **ArgoCD:** `kubectl get app terminus-portal-dev -n argocd -w` — watch for `Synced/Healthy`
  - [ ] Verify portal UI at `https://portal-dev.trantor.internal` — portal2 React app loads, services display, theme toggle works

- [x] Task 8: Production release — merge `develop` to `main` and verify (AC: #8)
  - [ ] Create PR: `develop` → `main` in the portal repo
  - [ ] Verify `ci.yml` passes before merging
  - [ ] Merge PR → `release.yml` fires for prod
  - [ ] Monitor: Temporal workflow `release-terminus-portal-prod-{sha[:8]}`
  - [ ] Verify ArgoCD app `terminus-portal` reaches `Synced/Healthy`
  - [ ] Verify portal2 UI at `https://portal.trantor.internal` — React app replaces the old v1 static HTML

## Dev Notes

**Target repos:**
- Service repo (portal code): `TargetProjects/terminus/portal/terminus-portal` (remote: `github.com/electricm0nk/terminus-portal`)
- Infra repo: `terminus.infra` on `vault.trantor.internal` (remote: `github.com/electricm0nk/terminus.infra`)

**Service name:** `terminus-portal` (used as `{service}` throughout cicd.md)
**Namespace:** `terminus-portal` (prod), `terminus-portal-dev` (dev)

**No database, no app secrets:**
- `ProvisionDB: false` — no `db-provision-terminus-portal` activity
- No `ExternalSecret` for app credentials — the portal is a static SPA with no runtime secrets
- The only ExternalSecret needed is the GHCR pull secret (`ghcr-pull-secret`) which already exists at `secret/terminus/default/ghcr/pull-token`
- **Critical:** The Temporal `ReleaseWorkflow` calls `SeedSecrets` unconditionally. If no `seed-terminus-portal-secrets` template exists in Semaphore project 3, the workflow will fail with `no template with name "seed-terminus-portal-secrets"`. Create a no-op stub playbook and template.

**`release.yml` Temporal input format (PascalCase — critical per cicd.md cheat sheet):**
```json
{"ServiceName":"terminus-portal","Environment":"dev","ProvisionDB":false,"ImageTag":"<full-sha>"}
```

**Semaphore project mapping (from cicd.md):**
- All Temporal-triggered release templates → Project 3 (`terminus-infra`)
- Project 2 (`terminus-infra-ops`) is for manual cluster ops only — do NOT register here

**Existing dev pattern for `ignoreDifferences` on ExternalSecrets:**
The GHCR ExternalSecret in `terminus-portal-dev` namespace will need the `ignoreDifferences` block added to its ArgoCD Application manifest — see cicd.md cheat sheet pattern. Count the number of `data:` entries (1 entry = indices 0 only). Omitting this causes persistent `OutOfSync` for the infra app.

**ArgoCD dev app valueFiles override:**
The dev Helm app should specify `valueFiles: [values.yaml, values-dev.yaml]` so only `image.tag` and `ingress.host` are overridden, inheriting all other values from `values.yaml`.

**Do NOT use `syncPolicy.automated` on `terminus-portal-dev.yaml`** or `terminus-portal.yaml` — Temporal's `TriggerArgoCDSync` activity is the release gate. See cicd.md: "For release-managed workloads, omit `syncPolicy.automated`".

**Verify playbooks — hostname policy:**
Use `http://portal[-dev].trantor.internal` (external DNS, Traefik ingress path). Do NOT use `svc.cluster.local` or `kubectl exec` — Semaphore runner resolves external hostnames, not cluster-internal DNS.

**CI workflow (`ci.yml`) — test scope:**
`npm test` runs `vitest run` (unit + component tests). The `test:e2e` script runs Playwright against a live server — this is NOT appropriate for the CI gate. Do not add `npm run test:e2e` to `ci.yml`.

**Branch protection:**
- `main`: require status check `test` from `ci.yml`
- `develop`: same rule
- Both branches: require PR (no direct push)

**Reference:** `docs/cicd.md` — "Deploying a New Service — Step by Step" section (Steps 1–6)

### Project Structure Notes

New files (portal service repo `github.com/electricm0nk/terminus-portal`):
```
.github/workflows/
└── release.yml
```

New/modified files (`terminus.infra` repo `github.com/electricm0nk/terminus.infra`):
```
platforms/k3s/
├── helm/terminus-portal/
│   └── values-dev.yaml             (NEW)
├── argocd/apps/
│   ├── terminus-portal-dev-infra.yaml  (NEW)
│   ├── terminus-portal-dev.yaml        (NEW)
│   ├── terminus-portal-dev-ingress.yaml (NEW)
│   └── terminus-portal.yaml            (MODIFY: remove syncPolicy.automated)
├── k8s/terminus-portal-dev-ingress/
│   └── ingress.yaml                (NEW)
└── ansible/playbooks/
    ├── verify-terminus-portal-dev-deploy.yml (NEW)
    ├── verify-terminus-portal-deploy.yml     (NEW)
    └── seed-terminus-portal-secrets.yml      (NEW — no-op stub for Temporal SeedSecrets activity)
tofu/environments/semaphore/
└── templates.tf (MODIFY: add verify-terminus-portal-* and seed-terminus-portal-secrets templates)
```

### References

- `docs/cicd.md` — primary reference, especially "Deploying a New Service", cheat sheet (ignoreDifferences, Semaphore project mapping, PascalCase ReleaseInput, `do not use latest`)
- `docs/terminus/portal/portal2/architecture.md` — architecture decisions, deployment target (`portal.trantor.internal`, namespace `terminus-portal`)
- Story 1.2 — Dockerfile (already built; satisfies the image build prerequisite)
- Story 1.3 — Helm chart (already adapted; `deploy/helm/` is the source chart)
- Story 1.4 — `ci.yml` (already exists; `release.yml` is a separate workflow)

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List
