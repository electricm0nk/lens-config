# Story 1.1: Research and Pin Crossplane Provider Versions

Status: ready-for-dev

## Story

As a platform engineer,
I want provider package versions pinned to specific stable releases in the Crossplane Provider manifests,
so that the installation is deterministic and not subject to unintended upgrades between deploys.

## Context

This is the **BLOCKER story** for Epic 1 — it must be completed before any Provider manifests can be committed or any other story in Epics 1, 2, or 3 can proceed.

**Source constraint (Agent Consistency Rule 1 from [arch doc](../../docs/terminus/infra/crossplane/architecture.md)):**
> Never use `latest` tag for provider packages — always pin a specific version

**Adversarial finding #1 (pre-implementation blocker):**
> Provider versions are referenced without specific version tags. An implementation agent will produce non-deterministic manifests.

### Verified Versions (live from Upbound Marketplace — checked 2026-03-30)

| Provider | Pinned Version | Package Reference | Crossplane Compat |
|---|---|---|---|
| provider-kubernetes | **v1.2.2** | `xpkg.upbound.io/upbound/provider-kubernetes:v1.2.2` | 2.0+ |
| provider-helm | **v1.2.1** | `xpkg.upbound.io/upbound/provider-helm:v1.2.1` | 2.0+ |

**Source:** https://marketplace.upbound.io/providers/upbound/provider-kubernetes (v1.2.2, last changed 2026-03-24)  
**Source:** https://marketplace.upbound.io/providers/upbound/provider-helm (v1.2.1, last changed 2026-03-24)

> ⚠️ **Verify before implementing:** Confirm these are still the latest stable versions at implementation time. Check marketplace pages above.

### Registry Clarification

The correct registry for Upbound-managed providers is:
- **Use:** `xpkg.upbound.io/upbound/provider-{name}:{version}`  
- **Do NOT use:** `xpkg.crossplane.io/crossplane-contrib/...` — this is the legacy community registry referenced in older readme examples

Both providers are Upbound-signed, CVE-remediated, with 12-month support window (ending 2027-03-25).

### InjectedIdentity RBAC Requirement (pre-confirmed from docs)

The Upbound Marketplace docs for `provider-kubernetes` confirm that InjectedIdentity (in-cluster) mode requires an explicit ClusterRoleBinding:

```bash
SA=$(kubectl -n crossplane-system get sa -o name | grep provider-kubernetes | \
  sed -e 's|serviceaccount\/|crossplane-system:|g')
kubectl create clusterrolebinding provider-kubernetes-admin-binding \
  --clusterrole cluster-admin \
  --serviceaccount="${SA}"
```

This addresses adversarial finding #6 and feeds directly into Story 2.3. Document this finding in the architecture.md update (see AC below).

### Crossplane Core Version

The architecture uses `crossplane-stable/crossplane` Helm chart. Compatibility requires **Crossplane 2.0+** for both providers. The stable chart should satisfy this — confirm the installed chart version is ≥ 2.0.0 when writing the Application.yaml in Story 1.2.

## Acceptance Criteria

1. Verify `provider-kubernetes` current stable version is `v1.2.2` via Upbound Marketplace (or confirm newer if updated since 2026-03-30)
2. Verify `provider-helm` current stable version is `v1.2.1` via Upbound Marketplace (or confirm newer if updated since 2026-03-30)
3. Add a `## Pinned Versions` section to `docs/terminus/infra/crossplane/architecture.md` documenting both pinned versions, the full package reference strings, the registry used, and the verification date
4. Confirm both providers are compatible with Crossplane 2.0+ (already confirmed — document in arch)
5. Document the InjectedIdentity ClusterRoleBinding requirement for `provider-kubernetes` in architecture.md under a new `## RBAC Notes` section (pre-research from marketplace docs — addresses adversarial finding #6)
6. The `## Pinned Versions` section in architecture.md must include the full `spec.package` string for copy-paste use in Stories 2.1 and 2.2

## Tasks / Subtasks

- [ ] Task 1: Verify provider-kubernetes version (AC: #1)
  - [ ] Open https://marketplace.upbound.io/providers/upbound/provider-kubernetes
  - [ ] Confirm latest stable version (expected: v1.2.2 or newer)
  - [ ] Record full package reference: `xpkg.upbound.io/upbound/provider-kubernetes:{version}`
  - [ ] Check release notes for any breaking changes vs v1.2.2

- [ ] Task 2: Verify provider-helm version (AC: #2)
  - [ ] Open https://marketplace.upbound.io/providers/upbound/provider-helm
  - [ ] Confirm latest stable version (expected: v1.2.1 or newer)
  - [ ] Record full package reference: `xpkg.upbound.io/upbound/provider-helm:{version}`
  - [ ] Check release notes for any breaking changes vs v1.2.1

- [ ] Task 3: Update architecture.md with pinned versions (AC: #3, #6)
  - [ ] Append `## Pinned Versions` section to `docs/terminus/infra/crossplane/architecture.md`
  - [ ] Append `## RBAC Notes` section documenting InjectedIdentity ClusterRoleBinding requirement
  - [ ] Both sections must be machine-readable (table format with full package strings)

- [ ] Task 4: Validate package references are reachable (AC: #4, #5)
  - [ ] Confirm `xpkg.upbound.io` is accessible from the terminus k3s cluster network
  - [ ] (Optional) `crossplane xpkg pull xpkg.upbound.io/upbound/provider-kubernetes:{version}` to confirm package is pullable

## Dev Notes

### Architecture Constraints

From `docs/terminus/infra/crossplane/architecture.md`:
- **Registry decision:** Upbound Marketplace (`xpkg.upbound.io`) — no local mirror
- **Consistency Rule 1:** Never use `latest` — always pin
- **Consistency Rule 2:** Always set `revisionHistoryLimit` on providers (value: set to `1` in Stories 2.1/2.2)
- **Consistency Rule 4:** Required labels on all managed resources:
  ```yaml
  labels:
    app.kubernetes.io/part-of: crossplane
    terminus.io/managed-by: argocd
  ```

### Target Files Modified in This Story

Only `docs/terminus/infra/crossplane/architecture.md` is modified. No manifests are created in this story — those are Stories 1.2–3.4.

### Sections to Append to architecture.md

Add these two sections after the existing `## Architecture Validation` section:

```markdown
## Pinned Versions
<!-- Updated: {date} -->

| Provider | Version | Full Package Reference | Registry | Verified |
|---|---|---|---|---|
| provider-kubernetes | v1.2.2 | `xpkg.upbound.io/upbound/provider-kubernetes:v1.2.2` | Upbound Marketplace | 2026-03-30 |
| provider-helm | v1.2.1 | `xpkg.upbound.io/upbound/provider-helm:v1.2.1` | Upbound Marketplace | 2026-03-30 |

**Notes:**
- Crossplane core: use `crossplane-stable` Helm chart, confirm installed version ≥ 2.0.0
- Both providers are Upbound-signed, CVE-remediated, 12-month support (until 2027-03-25)
- Update these versions before re-implementing in a new environment

## RBAC Notes

### provider-kubernetes — InjectedIdentity ClusterRoleBinding

`InjectedIdentity` for `provider-kubernetes` is NOT zero-setup. The Crossplane
ServiceAccount for the provider pod requires an explicit ClusterRoleBinding to manage
Kubernetes objects cross-namespace.

**Required ClusterRoleBinding (homelab: cluster-admin):**
```bash
SA=$(kubectl -n crossplane-system get sa -o name | grep provider-kubernetes | \
  sed -e 's|serviceaccount\/|crossplane-system:|g')
kubectl create clusterrolebinding provider-kubernetes-admin-binding \
  --clusterrole cluster-admin \
  --serviceaccount="${SA}"
```

This is handled in Story 2.3. Do NOT skip — without this, `Object` managed resources
will fail to reconcile with permission errors.

### provider-helm — InjectedIdentity

`provider-helm` in-cluster does not require additional ClusterRoleBinding beyond
the default Crossplane installation. The Helm service account operates within the
cluster's existing RBAC without cross-namespace writes. Confirm during Story 2.2 smoke test.
```

### References

- [Source: docs/terminus/infra/crossplane/architecture.md — Agent Consistency Rules]
- [Source: docs/terminus/infra/crossplane/adversarial-review-report.md — Finding #1, #6]
- [Source: https://marketplace.upbound.io/providers/upbound/provider-kubernetes]
- [Source: https://marketplace.upbound.io/providers/upbound/provider-helm]
- [Upstream: https://github.com/crossplane-contrib/provider-kubernetes]
- [Upstream: https://github.com/crossplane-contrib/provider-helm]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (GitHub Copilot)

### Completion Notes List

_To be filled in by dev agent during implementation_

### File List

- `docs/terminus/infra/crossplane/architecture.md` — append `## Pinned Versions` and `## RBAC Notes`
