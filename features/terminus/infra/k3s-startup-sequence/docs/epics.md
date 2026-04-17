---
stepsCompleted: [step-01, step-02, step-03, step-04]
inputDocuments:
  - docs/terminus/infra/k3s-startup-sequence/architecture.md
  - docs/terminus/infra/k3s-startup-sequence/finalizeplan-review.md
  - docs/terminus/infra/k3s-startup-sequence/techplan-adversarial-review.md
feature: k3s-startup-sequence
track: tech-change
domain: terminus
service: infra
status: ready-for-dev
updated_at: "2026-04-19"
---

# k3s-startup-sequence — Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for the `k3s-startup-sequence` tech-change feature. The goal is to add deterministic ArgoCD sync-wave annotations to all Application objects in `platforms/k3s/argocd/apps/` so that pod load order is predictable and dependency ordering is enforced on cluster restart or full sync.

All changes are annotation-only edits to existing Application YAML files — no Helm values, chart modifications, or Kubernetes object recreations are required. The acceptance criterion is: **all ArgoCD Application objects carry the correct `argocd.argoproj.io/sync-wave` annotation and the kubectl wave-compliance command confirms correct ordering**.

---

## Requirements Inventory

### Functional Requirements

| FR | Requirement |
|----|-------------|
| FR-01 | `argocd.argoproj.io/sync-wave: "1"` added to `temporal-infra.yaml` (currently no annotation) |
| FR-02 | `argocd.argoproj.io/sync-wave: "1"` added to `fourdogs-central-infra.yaml` (currently no annotation) |
| FR-03 | `argocd.argoproj.io/sync-wave: "1"` added to `fourdogs-central-dev-infra.yaml` (currently no annotation) |
| FR-04 | `argocd.argoproj.io/sync-wave: "1"` added to `fourdogs-kaylee-agent-infra.yaml` (currently no annotation) |
| FR-05 | `argocd.argoproj.io/sync-wave: "1"` added to `fourdogs-kaylee-agent-dev-infra.yaml` (currently no annotation) |
| FR-06 | `argocd.argoproj.io/sync-wave: "1"` added to `terminus-portal-infra.yaml` (currently no annotation) |
| FR-07 | `argocd.argoproj.io/sync-wave: "5"` added to `semaphoreui.yaml` (currently no annotation) |
| FR-08 | `actions-runner.yaml` wave corrected from `"2"` to `"4"` |
| FR-09 | `fourdogs-central.yaml` wave corrected from `"1"` to `"6"` |
| FR-10 | `fourdogs-central-dev.yaml` wave corrected from `"1"` to `"6"` |
| FR-11 | `fourdogs-emailfetcher.yaml` wave corrected from `"1"` to `"6"` |
| FR-12 | `fourdogs-emailfetcher-dev.yaml` wave corrected from `"1"` to `"6"` |
| FR-13 | `fourdogs-kaylee-agent.yaml` wave corrected from `"1"` to `"6"` |
| FR-14 | `fourdogs-kaylee-agent-dev.yaml` wave corrected from `"1"` to `"6"` |
| FR-15 | `grafana.yaml` wave corrected from `"2"` to `"6"` |
| FR-16 | `grafana-dev.yaml` wave corrected from `"2"` to `"6"` |
| FR-17 | `inference-gateway.yaml` wave corrected from `"2"` to `"6"` |
| FR-18 | `inference-gateway-dev.yaml` wave corrected from `"2"` to `"6"` |
| FR-19 | `influxdb.yaml` wave corrected from `"2"` to `"6"` |
| FR-20 | `influxdb-dev.yaml` wave corrected from `"2"` to `"6"` |
| FR-21 | `prometheus.yaml` wave corrected from `"2"` to `"6"` |
| FR-22 | `prometheus-dev.yaml` wave corrected from `"2"` to `"6"` |
| FR-23 | `ollama.yaml` wave corrected from `"2"` to `"6"` |
| FR-24 | `terminus-portal.yaml` wave corrected from `"2"` to `"6"` |
| FR-25 | `terminus-portal-dev.yaml` wave corrected from `"2"` to `"6"` |
| FR-26 | `grafana-ingress.yaml` wave corrected from `"3"` to `"7"` |
| FR-27 | `grafana-dev-ingress.yaml` wave corrected from `"3"` to `"7"` |
| FR-28 | `inference-gateway-ingress.yaml` wave corrected from `"3"` to `"7"` |
| FR-29 | `inference-gateway-dev-ingress.yaml` wave corrected from `"3"` to `"7"` |
| FR-30 | `influxdb-ingress.yaml` wave corrected from `"3"` to `"7"` |
| FR-31 | `influxdb-dev-ingress.yaml` wave corrected from `"3"` to `"7"` |
| FR-32 | `prometheus-ingress.yaml` wave corrected from `"3"` to `"7"` |
| FR-33 | `terminus-portal-dev-ingress.yaml` wave corrected from `"3"` to `"7"` |
| FR-34 | `temporal-worker-ingress.yaml` wave corrected from `"4"` to `"7"` |
| FR-35 | `temporal-worker-dev-ingress.yaml` wave corrected from `"4"` to `"7"` |
| FR-36 | `monitoring-servicemonitors.yaml` wave corrected from `"3"` to `"8"` |

### Non-Functional Requirements

| NFR | Requirement |
|-----|-------------|
| NFR-01 | All changes are annotation-only — no Helm values, chart version, or Kubernetes object spec modifications |
| NFR-02 | Wave annotations are declared as string values (quoted integers) to match ArgoCD convention |
| NFR-03 | After ArgoCD syncs, kubectl wave-compliance verification command confirms all Application objects carry correct wave values |
| NFR-04 | Rollback is a `git revert` of the annotation commit — cluster state is unaffected by the revert |

### Technical Architecture Requirements

| TA | Requirement |
|----|-------------|
| TA-01 | ADR-001: `argocd.argoproj.io/sync-wave` is the sole ordering mechanism — no alternative patterns |
| TA-02 | ADR-002: All `-infra` namespace-prep apps move to wave 1 regardless of workload type |
| TA-03 | ADR-003: All workload apps consolidate to wave 6 — no intermediate workload waves |
| TA-04 | ADR-004: All ingress apps consolidate to wave 7 — after all workload pods are scheduled |
| TA-05 | Epic 1 (critical gap fixes) should be applied before Epic 2 to establish the baseline ordering first |
| TA-06 | Wave compliance command in story ACs serves as the post-merge smoke test |

### FR Coverage Map

| Epic | Stories | FRs Covered |
|------|---------|-------------|
| Epic 1: Critical Ordering Gaps | 1-1 | FR-01–FR-07, NFR-01, NFR-02, TA-01, TA-02, TA-06 |
| Epic 2: Wave Consolidation | 2-1 | FR-08–FR-36, NFR-01, NFR-02, NFR-03, NFR-04, TA-01, TA-03, TA-04, TA-06 |

---

## Epic List

### Epic 1: Critical Ordering Gaps
Add `sync-wave` annotations to the 7 Application objects that have no annotation at all. Without these annotations, `temporal-infra`, all four `fourdogs-*-infra` namespace-prep apps, `terminus-portal-infra`, and `semaphoreui` deploy at wave 0 alongside cluster primitives — racing with ESO and Traefik before they are ready.

**Acceptance:** kubectl wave-compliance command shows all 7 apps at their correct wave values (1 or 5).

### Epic 2: Wave Consolidation
Correct the wave values on 23 existing Application objects that have annotations but at incorrect values. This includes promoting actions-runner from wave 2 to wave 4 (after temporal-workers), promoting all workload apps from wave 1 or 2 up to wave 6, promoting all ingress apps from wave 3 or 4 up to wave 7, and promoting monitoring-servicemonitors from wave 3 to wave 8.

**Acceptance:** kubectl wave-compliance command shows all 23 apps at their correct wave values.
