# Story 3.3: Configure ArgoCD Sync Wave Ordering

Status: ready-for-dev

## Story

As a platform engineer,
I want to verify and validate the ArgoCD sync-wave ordering across all Crossplane Application manifests,
so that Crossplane components install in the correct dependency order: core → providers → providerconfigs.

## Context

Crossplane requires strict installation ordering enforced by ArgoCD sync waves:
- Wave 0: crossplane-core (Helm chart) — installs Crossplane CRDs and operator
- Wave 1: providers — Provider CRDs; Crossplane CRD must exist first
- Wave 2: providerconfigs — ProviderConfig CRDs; Provider must register the CRD first

ArgoCD will not advance to a higher wave until all lower-wave resources report `Healthy`.
Without this ordering, ArgoCD sync fails with `no matches for kind` errors.

**Adversarial Findings Addressed:**
- **Finding #9:** Sync wave ordering omitted from architecture — this story validates it end-to-end
- **Finding #11:** `selfHeal: true` and `prune: true` — confirmed on all Applications in this story

**Depends-on:** Stories 1.2, 2.1, 3.1 (Application.yaml files must exist to audit)

## Acceptance Criteria

1. All Application.yaml files have correct sync-wave annotations:
   - `crossplane-core/Application.yaml`: `argocd.argoproj.io/sync-wave: "0"`
   - `providers/Application.yaml`: `argocd.argoproj.io/sync-wave: "1"`
   - `providerconfigs/Application.yaml`: `argocd.argoproj.io/sync-wave: "2"`
2. All Application.yaml files have `spec.syncPolicy.automated: {prune: true, selfHeal: true}` (finding #11)
3. All Application.yaml files have `spec.syncPolicy.syncOptions` including `ServerSideApply=true`
4. Wave ordering is validated in cluster: observe ArgoCD sync and confirm wave progression
5. Architecture.md `## Deferred Decisions` table updated with the final sync-wave configuration decision (closing the deferred item)
6. No Application.yaml has `selfHeal: false` or `prune: false` explicitly set

## Tasks / Subtasks

- [ ] Task 1: Audit all Application.yaml sync-wave annotations (AC: #1)
  - [ ] `grep -r "sync-wave" terminus.infra/apps/crossplane/`
  - [ ] Verify: crossplane-core=0, providers=1, providerconfigs=2
  - [ ] Correct any wrong values

- [ ] Task 2: Audit all Application.yaml syncPolicy settings (AC: #2–#3, #6)
  - [ ] `grep -r "selfHeal\|prune\|ServerSideApply" terminus.infra/apps/crossplane/`
  - [ ] Confirm all three Application.yaml files have `prune: true`, `selfHeal: true`, `ServerSideApply=true`
  - [ ] Correct any missing settings

- [ ] Task 3: Validate wave progression in ArgoCD (AC: #4)
  - [ ] Trigger a full sync of the root crossplane Application (or the App-of-Apps parent)
  - [ ] Observe: crossplane-core Application syncs first and reaches Healthy
  - [ ] Observe: providers Application begins syncing only after crossplane-core is Healthy
  - [ ] Observe: providerconfigs Application begins syncing only after providers are Healthy
  - [ ] Document observed wave progression in story completion notes

- [ ] Task 4: Update architecture.md (AC: #5)
  - [ ] Find the `Deferred Decisions` table in architecture.md
  - [ ] Add a final row or note confirming sync-wave ordering is resolved:
    ```
    | ArgoCD sync-wave ordering | Resolved in Story 3.3 — wave 0/1/2 |
    ```

## Dev Notes

### Wave Mechanics

ArgoCD evaluates sync waves during a sync operation:
1. All resources with wave `"0"` are applied and ArgoCD waits for `Healthy`
2. Once wave 0 is `Healthy`, ArgoCD applies wave `"1"` resources
3. Once wave 1 is `Healthy`, wave `"2"` resources are applied

"Healthy" for an ArgoCD Application means the Application's `status.health.status == Healthy`,
which means Crossplane core is running and providers have their pods up.

### Sync Wave on Application vs CRD Manifests

The sync-wave annotation can be placed on either:
- The **Application** resource (what we're doing) — controls when ArgoCD processes the Application
- The **resources inside** an Application — controls ordering within a single Application

We annotate the **Application resources** because they are all in the same App-of-Apps scope.
The Provider and ProviderConfig CRD manifests themselves do NOT need wave annotations
(they're managed by their respective Applications).

### ServerSideApply Rationale

Crossplane installs cluster-scoped CRDs with complex field managers. ArgoCD's default
client-side apply conflicts with Crossplane's field ownership. `ServerSideApply=true` resolves
this by delegating apply semantics to the Kubernetes API server.

Without it: `Apply failed with 1 conflict(s): conflict with "crossplane" ...`

### Architecture Update Location

In `docs/terminus/infra/crossplane/architecture.md`, find:
```markdown
| Decision | Deferred Until |
```
Add/update the sync-wave row to mark it resolved.

### References

- [Source: docs/terminus/infra/crossplane/architecture.md — Deferred Decisions, Project Structure]
- [Source: docs/terminus/infra/crossplane/adversarial-review-report.md — Findings #9, #11]
- [Upstream: https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/]
- [Upstream: https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (GitHub Copilot)

### Completion Notes List

_To be filled in by dev agent during implementation_

### File List

- `terminus.infra/apps/crossplane/*/Application.yaml` — modify if sync-wave corrections needed
- `docs/terminus/infra/crossplane/architecture.md` — append sync-wave resolution note
