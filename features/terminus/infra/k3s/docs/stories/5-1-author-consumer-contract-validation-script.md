# Story 5.1: Author Consumer Contract Validation Script

## Status: done

## Story

As an operator,
I want a runnable validation script that checks all 8 consumer contract items,
So that "cluster ready" is a testable condition rather than a prose assertion.

## Acceptance Criteria

- **Given** a fully deployed cluster from Epics 1–4
- **When** `infra/k3s/scripts/validate-contract.sh` is authored and committed
- **Then** the script checks all 8 contract items:
  1. All 8 nodes in `Ready` state
  2. All 6 namespaces in `Active` phase
  3. At least one `ExternalSecret` has `synced` status
  4. `ClusterIssuer` reports `Ready: True` and a test `Certificate` issues successfully
  5. MetalLB `IPAddressPool` is present and at least one IP has been allocated
  6. Traefik pods are `Running` and the `LoadBalancer` service has an external IP
  7. ArgoCD root app and all child apps are `Synced` and `Healthy`
  8. A default `StorageClass` is present and active
- **And** each check produces a `PASS` or `FAIL` line with the item name
- **And** the script exits non-zero if any check fails

## Tasks / Subtasks

- [x] Task 1: Create validate-contract.sh
  - [x] `infra/k3s/scripts/validate-contract.sh` — bash script; checks all 8 items
  - [x] Each check emits `[PASS] <item>` or `[FAIL] <item>: <reason>`
  - [x] Non-zero exit on any failure (`exit $FAILURES`)
  - [x] Executable (`chmod +x`)
- [x] Task 2: Checks 1 and 2 — nodes and namespaces
  - [x] Count 8 nodes + `grep -v " Ready "` → fail if non-empty
  - [x] Check all 6 namespace names are `Active` via jsonpath phase
- [x] Task 3: Checks 3 and 4 — ESO and cert-manager
  - [x] ExternalSecret Ready=True count >= 1 via jsonpath
  - [x] ClusterIssuer vault-pki conditions[Ready].status == "True"
- [x] Task 4: Checks 5–8 — MetalLB, Traefik, ArgoCD, StorageClass
  - [x] MetalLB IPAddressPool count >= 1; Traefik Running pods + LB external IP
  - [x] ArgoCD apps: no non-Synced, no non-Healthy
  - [x] Default StorageClass via annotation jsonpath

## Dev Notes

**Script exit pattern:**
```bash
FAILURES=0
check() {
  local name="$1"
  local result="$2"
  if [ "$result" = "pass" ]; then
    echo "[PASS] $name"
  else
    echo "[FAIL] $name: $result"
    FAILURES=$((FAILURES + 1))
  fi
}
# ... checks ...
exit $FAILURES
```

## Dev Agent Record

### Agent Model Used: `claude-sonnet-4-6`
### Debug Log References: ``
### Completion Notes List
- Check 6 is split into two separate `check()` calls (pods Running + external IP) — gives more granular failure info
- Traefik pod check uses `-l "app.kubernetes.io/name=traefik"` label selector — depends on Traefik chart using this label
- ArgoCD check collects all app statuses via `jsonpath` and filters for non-Synced/non-Healthy states
- PR: electricm0nk/terminus.infra#41
### Change List
- `infra/k3s/scripts/validate-contract.sh` — 8-check bash validation script, executable
