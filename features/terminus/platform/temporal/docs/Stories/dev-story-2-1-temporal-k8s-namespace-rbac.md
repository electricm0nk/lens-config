# Story 2.1: Temporal k8s Namespace + RBAC

Status: done

## Story

As a platform developer,
I want a dedicated `temporal` Kubernetes namespace and worker ServiceAccount with appropriate RBAC established in the cluster before any Temporal resources are applied,
so that all Temporal server and worker resources are isolated and synced in the correct ArgoCD wave order.

## Acceptance Criteria

1. `k8s/namespace.yaml` created: `apiVersion: v1, kind: Namespace, metadata: {name: temporal}`
2. `k8s/rbac-worker.yaml` created with:
   - `ServiceAccount` in `temporal` namespace: `temporal-worker`
   - `Role` in `temporal` namespace: read access to `temporal` namespace secrets (for worker to read ESO-synced secrets)
   - `RoleBinding` binding `temporal-worker` ServiceAccount to the Role
3. Both files have ArgoCD sync wave annotation: `argocd.argoproj.io/sync-wave: "1"`
4. Files pass `kubectl apply --dry-run=client -f <file>` against the k3s cluster
5. `k8s/` directory contains only these two files — no Helm overrides or application configs

## Tasks / Subtasks

- [ ] Task 1: Create `k8s/namespace.yaml` (AC: 1, 3, 4)
  - [ ] Create `k8s/namespace.yaml`:
    ```yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: temporal
      annotations:
        argocd.argoproj.io/sync-wave: "1"
    ```
  - [ ] Validate: `kubectl apply --dry-run=client -f k8s/namespace.yaml`
  - [ ] Commit: `feat(k8s): add temporal namespace manifest`

- [ ] Task 2: Create `k8s/rbac-worker.yaml` (AC: 2, 3, 4)
  - [ ] Create `k8s/rbac-worker.yaml` with three documents (separated by `---`):
    - `ServiceAccount`: name `temporal-worker`, namespace `temporal`, sync-wave `"1"`
    - `Role`: name `temporal-worker-role`, namespace `temporal`, rules: `get/list/watch` on `secrets`
    - `RoleBinding`: name `temporal-worker-binding`, namespace `temporal`, binds SA to Role
  - [ ] All three resources have `argocd.argoproj.io/sync-wave: "1"` annotation
  - [ ] Validate: `kubectl apply --dry-run=client -f k8s/rbac-worker.yaml`
  - [ ] Commit: `feat(k8s): add temporal-worker ServiceAccount and RBAC`

- [ ] Task 3: Verify k8s directory hygiene (AC: 5)
  - [ ] Confirm `k8s/` contains only `namespace.yaml` and `rbac-worker.yaml`
  - [ ] No Helm values, Application CRs, or other configs in `k8s/`

## Dev Notes

### Target Repo Path
```
TargetProjects/terminus/platform/terminus.platform/
```

### Architecture Reference
Source: Architecture Step 6 (`k8s/namespace.yaml`, `k8s/rbac-worker.yaml`), N8 (boundary separation)

### Sync Wave Strategy
Wave `"1"` — namespace and RBAC must exist before ESO ExternalSecrets (Story 2.2) try to sync. Wave string is quoted per Kubernetes annotation conventions.

### Role vs ClusterRole Decision
Use `Role` scoped to `temporal` namespace (principle of least privilege). The `temporal-worker` ServiceAccount only needs access to secrets within the `temporal` namespace — no cluster-wide access required.

### Constitutional Override — Develop-First Integration
Implementation work lands on `develop` in `terminus.platform` (Terminus Art. 5 override active).

### Not In Scope
- ESO ExternalSecrets (Story 2.2)
- Helm chart or ArgoCD Application manifests (Stories 2.3, 3.1)
- Any Temporal server or worker configuration

## Pre-Implementation Constitution Gates

| Gate | Article | Result |
|------|---------|--------|
| Track declared | Org Art. 1 | ✅ PASS — `tech-change` |
| Architecture documented | Org Art. 3 | ✅ PASS — `docs/terminus/platform/temporal/architecture.md` |
| No confidential data | Org Art. 4 | ✅ PASS — no secrets or credentials |
| Git discipline | Org Art. 5 | ✅ PASS — develop-first integration authorized by terminus Art. 5 |
| Security first | Org Art. 9 | ✅ PASS — least-privilege Role scoped to namespace |
| Repo as source of truth | Org Art. 10 | ✅ PASS — all context from committed docs |
| Develop-first integration | Terminus Art. 5 | ✅ OVERRIDE ACTIVE — land implementation work on `develop` in `terminus.platform` |

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (copilot)

### Debug Log References

### Completion Notes List

### File List
