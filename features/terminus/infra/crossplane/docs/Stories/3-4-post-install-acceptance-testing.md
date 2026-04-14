# Story 3.4: Post-Install Acceptance Testing

Status: ready-for-dev

## Story

As a platform engineer,
I want to run a defined post-install acceptance test sequence against the Crossplane installation,
so that I have documented and reproducible proof that the installation is healthy and functional
before the initiative is marked complete.

## Context

The adversarial reviewer (Finding #10) flagged that no acceptance criteria or done criteria
were defined for the overall initiative. This story closes that gap by defining and executing
a structured validation sequence covering:

1. Crossplane core operator health
2. Provider pod health and HEALTHY status
3. CRD registration completeness
4. ProviderConfig SYNCED status
5. Smoke tests: one Object (kubernetes provider), one Release (helm provider, optional)
6. ArgoCD Application health (all 3 Applications Synced+Healthy)

Results are documented in `docs/terminus/infra/crossplane/acceptance-results.md` (new file).

**Adversarial Findings Addressed:**
- **Finding #10:** No acceptance criteria defined — this story is the acceptance gate for the entire initiative

**Depends-on:** All prior stories (3.1, 3.2, 2.3, especially — providers and providerconfigs must be installed)

## Acceptance Criteria

1. All Crossplane pods in `crossplane-system` are `Running` / `Ready`
2. Both providers (`provider-kubernetes`, `provider-helm`) reach `HEALTHY: True`
3. All Crossplane CRDs are registered: `kubectl get crds | grep crossplane.io` returns expected CRDs
4. Both ProviderConfigs show `SYNCED: True` and `READY: True`
5. Object smoke test succeeds (provider-kubernetes): create a ConfigMap via Object manifest, confirm it appears in cluster
6. ArgoCD reports all 3 Applications (crossplane-core, providers, providerconfigs) as `Synced` and `Healthy`
7. Acceptance results documented in `docs/terminus/infra/crossplane/acceptance-results.md`
8. `docs/implementation-artifacts/sprint-status.yaml` story 3.4 updated to `complete` after testing passes

## Tasks / Subtasks

- [ ] Task 1: Verify Crossplane core operator pods (AC: #1)
  ```bash
  kubectl -n crossplane-system get pods
  # Expected: crossplane-*, crossplane-rbac-manager-* all Running/Ready
  ```
  Record pod names and status in acceptance-results.md.

- [ ] Task 2: Verify Provider health (AC: #2)
  ```bash
  kubectl get providers
  # Expected: provider-kubernetes HEALTHY=True, provider-helm HEALTHY=True
  # Wait up to 5 minutes for providers to pull and start
  ```
  Record provider versions and HEALTHY status in acceptance-results.md.

- [ ] Task 3: Verify CRD registration (AC: #3)
  ```bash
  kubectl get crds | grep crossplane.io
  # Minimum expected CRDs:
  # compositeresourcedefinitions.apiextensions.crossplane.io
  # compositions.apiextensions.crossplane.io
  # objects.kubernetes.crossplane.io      (from provider-kubernetes)
  # releases.helm.crossplane.io           (from provider-helm)
  # providerconfigs.kubernetes.crossplane.io
  # providerconfigs.helm.crossplane.io
  ```
  Record CRD list in acceptance-results.md.

- [ ] Task 4: Verify ProviderConfig sync status (AC: #4)
  ```bash
  kubectl get providerconfigs.kubernetes.crossplane.io
  kubectl get providerconfigs.helm.crossplane.io
  # Expected: both show SYNCED=True, READY=True
  ```
  Record ProviderConfig status in acceptance-results.md.

- [ ] Task 5: Object smoke test — kubernetes provider (AC: #5)
  ```bash
  # Create a test Object manifest:
  cat <<EOF | kubectl apply -f -
  apiVersion: kubernetes.crossplane.io/v1alpha2
  kind: Object
  metadata:
    name: crossplane-smoke-test
  spec:
    providerConfigRef:
      name: default
    forProvider:
      manifest:
        apiVersion: v1
        kind: ConfigMap
        metadata:
          name: crossplane-smoke-test-cm
          namespace: default
        data:
          test: "acceptance-passed"
  EOF

  # Verify Object is READY=True
  kubectl get object crossplane-smoke-test
  # Verify ConfigMap was created
  kubectl get configmap crossplane-smoke-test-cm -n default

  # Cleanup
  kubectl delete object crossplane-smoke-test
  ```
  Record test outcome (pass/fail) in acceptance-results.md.
  If Object CRD apiVersion differs, check: `kubectl api-resources | grep object`

- [ ] Task 6: ArgoCD Application health check (AC: #6)
  ```bash
  # Via CLI:
  argocd app list | grep crossplane
  # Expected: crossplane-core Synced Healthy, providers Synced Healthy, providerconfigs Synced Healthy

  # OR via kubectl:
  kubectl get applications -n argocd | grep crossplane
  ```
  Record Application health in acceptance-results.md.

- [ ] Task 7: Create acceptance-results.md (AC: #7)
  - [ ] Create `docs/terminus/infra/crossplane/acceptance-results.md`
  - [ ] Document: test date, cluster version, Crossplane version, all task results
  - [ ] Include: pass/fail verdict with any caveats

- [ ] Task 8: Update sprint-status.yaml (AC: #8)
  - [ ] Mark story `3-4-post-install-acceptance-testing: complete`
  - [ ] If all epics complete: mark all epics `complete` and set initiative closed flag in crossplane.yaml

## Dev Notes

### Provider HEALTHY Timing

After ProviderConfig is applied, providers need to:
1. Pull the provider package from Upbound Marketplace
2. Start the provider pod
3. Register the provider's CRDs with the API server
4. Report HEALTHY

This can take 2–5 minutes depending on registry pull speed. Use:
```bash
kubectl get providers -w
```
to watch until both providers report `HEALTHY: True`.

### Object apiVersion Note

The `Object` CRD for provider-kubernetes v1.2.2 may use `v1alpha2` or `v1alpha1`.
Check with:
```bash
kubectl api-resources | grep kubernetes.crossplane.io
```
Use the version listed in the output for the smoke test manifest.

### Release Smoke Test (Optional)

If a helm provider smoke test is desired, an Object is not needed — use a `Release` manifest:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: helm.crossplane.io/v1beta1
kind: Release
metadata:
  name: crossplane-smoke-test-helm
spec:
  providerConfigRef:
    name: default
  forProvider:
    chart:
      name: podinfo
      repository: https://stefanprodan.github.io/podinfo
      version: "6.7.0"
    namespace: default
    skipCreateNamespace: true
EOF
kubectl get release crossplane-smoke-test-helm
kubectl delete release crossplane-smoke-test-helm
```
The Release smoke test is optional — document the decision in acceptance-results.md.

### Acceptance Results File Structure

```markdown
# Crossplane Acceptance Test Results

**Date:** YYYY-MM-DD
**Cluster:** terminus-k3s
**Crossplane Version:** v1.x.x (from `kubectl get pods -n crossplane-system -o jsonpath=...`)
**Tester:** [name]

## Results

| Test | Status | Notes |
|------|--------|-------|
| Core pods Running | PASS/FAIL | ... |
| provider-kubernetes HEALTHY | PASS/FAIL | version confirmed: v1.2.2 |
| provider-helm HEALTHY | PASS/FAIL | version confirmed: v1.2.1 |
| CRDs registered | PASS/FAIL | X CRDs found |
| ProviderConfig kubernetes SYNCED | PASS/FAIL | ... |
| ProviderConfig helm SYNCED | PASS/FAIL | ... |
| Object smoke test | PASS/FAIL | ... |
| ArgoCD crossplane-core Synced+Healthy | PASS/FAIL | ... |
| ArgoCD providers Synced+Healthy | PASS/FAIL | ... |
| ArgoCD providerconfigs Synced+Healthy | PASS/FAIL | ... |

## Verdict

PASS / FAIL — [brief summary]

## Caveats / Known Issues

_None_ OR [list items]
```

### Initiative Closure

When all stories in sprint-status.yaml are `complete`, update:
- `_bmad-output/lens-work/initiatives/terminus/infra/crossplane.yaml`
  - Set `current_phase: complete`
  - Set `dev_phase: complete`
- This is the formal signal to LENS that the initiative is closed.

### References

- [Source: docs/terminus/infra/crossplane/epics.md — Epic 3, Story 3.4]
- [Source: docs/terminus/infra/crossplane/adversarial-review-report.md — Finding #10]
- [Source: docs/implementation-artifacts/2-3-validate-injectedidentity-rbac-for-provider-kubernetes.md — RBAC setup]
- [Upstream: https://docs.crossplane.io/latest/getting-started/provider-kubernetes/]
- [Upstream: https://docs.crossplane.io/latest/getting-started/provider-helm/]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (GitHub Copilot)

### Completion Notes List

_To be filled in by dev agent during implementation_

### File List

- `docs/terminus/infra/crossplane/acceptance-results.md` — CREATE NEW
- `docs/implementation-artifacts/sprint-status.yaml` — update story 3.4 to complete
- `_bmad-output/lens-work/initiatives/terminus/infra/crossplane.yaml` — update if all stories complete
