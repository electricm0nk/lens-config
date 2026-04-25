---
feature: gpu-passthrough
doc_type: adversarial-review
phase: techplan
source: phase-complete
verdict: pass-with-warnings
reviewed_artifacts:
  - architecture.md
updated_at: "2026-04-25T00:00:00Z"
---

# TechPlan Adversarial Review - GPU Passthrough to k3s

Feature: gpu-passthrough  
Phase: techplan  
Verdict: pass-with-warnings

## Findings

### HIGH

T-1 - IOMMU isolation remains a hard blocker until proven

The architecture now defines a hard gate, but no concrete host evidence is recorded yet for the RTX 3090 group layout. If the card shares a group with critical host devices, rollout cannot proceed safely.

Recommendation:
- Require captured IOMMU group output as a pre-implementation artifact.
- Mark implementation blocked until this evidence exists.

T-2 - GPU worker bootstrap path is underspecified

The architecture correctly targets a k3s worker but does not lock down one bootstrap method for NVIDIA driver + runtime + k3s compatibility (cloud-init vs Ansible vs image baking). Drift risk is high without one canonical path.

Recommendation:
- Pick one bootstrap strategy and define version pinning requirements (driver, container toolkit, k3s runtime).

### MEDIUM

T-3 - Routing config coupling needs explicit test gate

Gateway/provider-routing is the right integration point, but there is no explicit end-to-end acceptance gate proving fallback works when the local provider is down.

Recommendation:
- Add acceptance criteria: local provider healthy path and forced-failure fallback path must both pass.

T-4 - Node isolation policy exists but resource pressure behavior is not defined

Taints/labels are present, but no policy is stated for CPU/memory pressure on the GPU worker. Inference pods may starve under load.

Recommendation:
- Define resource requests/limits and eviction expectations for inference workloads.

T-5 - MAC system-of-record is still implicit

Required MAC pinning is a good correction, but the authoritative source for assigned MAC values is not declared.

Recommendation:
- Declare one source of truth (tfvars in repo or Vault record) and enforce it in runbook/checklist.

T-6 - Security boundary relies on intent, not explicit policy

Architecture says provider is internal-only, but does not require NetworkPolicy or equivalent control.

Recommendation:
- Add mandatory namespace/pod-level network policy so only gateway components can call local provider service.

### LOW

T-7 - Single-GPU operational limits are not explicit

RTX 3090 has no MIG. Concurrency and queueing assumptions are not yet documented.

Recommendation:
- Define target concurrency envelope and model memory budget in implementation readiness.

T-8 - Observability handoff is underdefined

The design relies on gateway/provider health, but GPU-specific telemetry requirements are not explicit.

Recommendation:
- Add minimal telemetry gate: node GPU visibility, pod GPU allocation, and gateway degraded-state counters.

## Party-Mode Challenge Summary

Network perspective:
- Keep provider traffic in-cluster and avoid raw LAN exposure.

Security perspective:
- Enforce gateway-only access with NetworkPolicy, not convention.

Platform Ops perspective:
- The architecture is directionally correct, but first-run success depends on deterministic worker bootstrap and validated rollback checks.

## Blind-Spot Questions to Carry Into FinalizePlan

1. Which exact Proxmox node and PCI ID will be used for the 3090 first rollout?
2. What is the canonical bootstrap method for NVIDIA runtime on gpu-worker-01?
3. Where is the MAC address source of truth recorded and reviewed?
4. Which workload classes become local-first once Ollama is live?
5. What is the minimum fallback test that must pass before rollout sign-off?

## Verdict

pass-with-warnings

Rationale:
- The architecture now aligns with the stated product intent: gateway on k3s consuming local GPU via k3s.
- No fatal design contradiction remains.
- Warnings are execution-readiness risks, not architecture invalidation.