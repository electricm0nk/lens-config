# Story 1.3: Create crossplane-core values.yaml

Status: ready-for-dev

## Story

As a platform engineer,
I want a `values.yaml` file for the Crossplane Helm chart that explicitly documents which defaults are accepted and which are overridden,
so that the installation is auditable and any future tuning changes are tracked in git.

## Context

The arch document lists `values.yaml` as "Helm values override (if any)" with no further guidance.
Adversarial finding #4 flagged this as undefined. This story resolves it by creating a
minimal, well-commented values.yaml that explicitly documents each field decision — override or accept-default.

**Depends-on:** Story 1.2 (Application.yaml references `valuesFiles: [values.yaml]`)  
**Target file:** `terminus.infra/apps/crossplane/crossplane-core/values.yaml`

### Adversarial Findings Addressed

- **Finding #4:** values.yaml content was undefined — this story creates it with explicit rationale for each field
- **Finding #5:** `revisionHistoryLimit` value was unstated — this story sets it to `1`

## Acceptance Criteria

1. File `terminus.infra/apps/crossplane/crossplane-core/values.yaml` exists and is valid YAML
2. `revisionHistoryLimit: 1` is explicitly set (Agent Consistency Rule 2 — homelab: 1 is sufficient)
3. File includes a comment for each field: either the override value with rationale, or `# using upstream default` with the default value noted
4. File passes `helm template crossplane crossplane-stable/crossplane -f values.yaml` without errors
5. No secrets or credentials are present in the file (Agent Consistency Rule 5)

## Tasks / Subtasks

- [ ] Task 1: Research available Chart values (AC: #3)
  - [ ] Run `helm show values crossplane-stable/crossplane --version <pinned-version>` to see full values schema
  - [ ] Identify fields relevant to homelab deployment: resource limits, revisionHistoryLimit, metrics, leader election

- [ ] Task 2: Write values.yaml with explicit field decisions (AC: #1–#4)
  - [ ] Use the template in Dev Notes as a starting point
  - [ ] Fill in any values discovered in Task 1 that need tuning for homelab

- [ ] Task 3: Validate with helm template (AC: #4)
  - [ ] `helm repo add crossplane-stable https://charts.crossplane.io/stable`
  - [ ] `helm template crossplane crossplane-stable/crossplane --version <pinned-version> -f values.yaml`

## Dev Notes

### Target Repository

All files created in this story go into the **`terminus.infra`** repo.
Path: `terminus.infra/apps/crossplane/crossplane-core/values.yaml`

### Template

```yaml
# Crossplane Helm chart values — terminus homelab
# Chart: crossplane-stable/crossplane
# Pinned version: see architecture.md ## Pinned Versions
# Last updated: 2026-03-30

# revisionHistoryLimit: set to 1 to prevent unbounded revision accumulation.
# Agent Consistency Rule 2: always set this. Homelab — 1 is sufficient.
# Upstream default: 10
revisionHistoryLimit: 1

# resourcesCrossplane: using upstream defaults for homelab.
# Single-node k3s cluster — resource constraints not required.
# Upstream defaults: requests cpu=100m,memory=256Mi; limits cpu=500m,memory=512Mi
# resourcesCrossplane:
#   limits:
#     cpu: 500m
#     memory: 512Mi
#   requests:
#     cpu: 100m
#     memory: 256Mi

# metrics: disabled — no Prometheus stack in homelab at this stage.
# Upstream default: disabled
# metrics:
#   enabled: false

# leaderElection: enabled (upstream default) — single replica, harmless.
# Upstream default: enabled

# provider: no pre-installed providers via values — installed via Provider CRDs in Story 2.1/2.2.
# This keeps provider version management in git (Provider CRDs) not helm values.
```

### Why Minimal Overrides

For a homelab single-node k3s cluster:
- No memory pressure requiring resource limits adjustment
- No metrics stack requiring Prometheus scraping
- Revision history of 1 is sufficient (no need to rollback > 1 versions)
- Leader election works fine with single replica

Resist the urge to add overrides "just in case" — every override requires future maintenance.

### references

- [Source: docs/terminus/infra/crossplane/architecture.md — Consistency Rules]
- [Source: docs/terminus/infra/crossplane/adversarial-review-report.md — Findings #4, #5]
- [Upstream: https://artifacthub.io/packages/helm/crossplane/crossplane — values schema]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (GitHub Copilot)

### Completion Notes List

_To be filled in by dev agent during implementation_

### File List

- `terminus.infra/apps/crossplane/crossplane-core/values.yaml` — create
