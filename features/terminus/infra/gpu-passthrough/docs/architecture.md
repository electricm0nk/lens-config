---
feature: gpu-passthrough
doc_type: architecture
status: draft
goal: "Pass through the Gigabyte RTX 3090 Turbo from Proxmox into a dedicated k3s worker VM so the live inference gateway can route inference workloads to local GPU capacity"
key_decisions:
  - ADR-1: GPU consumer is a dedicated k3s worker VM, not a standalone inference VM
  - ADR-2: Static VFIO host binding plus explicit IOMMU gate before provisioning
  - ADR-3: Extend k3s OpenTofu patterns with a dedicated gpu-worker module and required MAC pinning
  - ADR-4: Ollama is the initial local GPU provider; gateway/provider-routing handles policy and fallback
  - ADR-5: GPU worker is node-pinned and tainted for inference-only workloads
open_questions:
  - OQ-1: Exact Proxmox node and PCI address for the RTX 3090 Turbo
  - OQ-2: Final static IP and DNS record for the GPU worker
  - OQ-3: MIG support is unavailable on 3090; define concurrency targets and queueing policy
depends_on:
  - terminus-infra-proxmox
  - terminus-inference-gateway
  - terminus-inference-provider-routing
blocks: []
updated_at: "2026-04-25T00:00:00Z"
---

# Architecture Decision Document - GPU Passthrough to k3s

Feature: gpu-passthrough  
Track: tech-change  
Domain: terminus/infra

## Overview

This architecture passes the physical Gigabyte RTX 3090 Turbo through Proxmox VFIO into a dedicated k3s worker VM. The live inference gateway in k3s remains the control plane. It routes eligible workloads to a local Ollama provider running on the GPU worker.

This replaces the earlier standalone inference VM concept.

## Goal

Expose local GPU acceleration to the existing inference gateway by making GPU capacity schedulable inside k3s, while preserving clear routing, fallback, and policy boundaries already implemented in gateway/provider-routing.

## Scope

In scope:
- Proxmox host IOMMU + VFIO setup for RTX 3090 passthrough
- Dedicated GPU k3s worker VM provisioning with pinned MAC
- Kubernetes NVIDIA runtime/device plugin enablement
- Ollama deployment on GPU worker
- Inference gateway route configuration to local GPU provider
- Node taints/labels and scheduling guards for inference-only GPU usage

Out of scope:
- Multi-GPU orchestration
- Model registry and lifecycle tooling
- Cross-cluster GPU scheduling
- External/public ingress for raw Ollama endpoint

## System Design

### Logical flow

1. Proxmox host exposes RTX 3090 via VFIO.
2. OpenTofu provisions a dedicated k3s worker VM with GPU passthrough.
3. Worker boots with NVIDIA driver/container runtime support.
4. NVIDIA device plugin advertises nvidia.com/gpu in cluster.
5. Ollama deployment requests GPU resources on the dedicated worker.
6. Inference gateway provider-routing sends local-eligible traffic to Ollama service.
7. If local provider is unhealthy/exhausted, gateway falls back per existing routing policy.

### Architecture diagram

```
Client -> terminus-inference-gateway (k3s)
         -> provider-routing profile selection
         -> local provider service (Ollama, k3s)
         -> Pod scheduled on gpu-worker-01 (nvidia.com/gpu)

Proxmox host (node pinned)
  -> VFIO binds RTX 3090
  -> GPU passed through to gpu-worker-01 VM
```

## Infrastructure Architecture

### Proxmox host prerequisites

Required before provisioning:
- VT-d/AMD-Vi enabled in BIOS
- Secure Boot disabled for host kernel path used by VFIO
- Kernel params enabled:
  - Intel: intel_iommu=on iommu=pt
  - AMD: amd_iommu=on iommu=pt
- VFIO modules loaded at boot (vfio, vfio_pci, vfio_iommu_type1)
- RTX 3090 vendor:device bound to vfio-pci

Hard go/no-go gate:
- If GPU IOMMU group includes critical host devices (storage/NIC), do not proceed until resolved by slot change or hardware remap.
- ACS override is a last resort and must be explicitly approved in implementation checklist.

### OpenTofu shape

Create a dedicated module (or specialized k3s extension module) for gpu-worker:

Required inputs:
- target_node
- vm_id
- gpu_pci_id
- mac_address
- static_ipv4
- kube_node_name

Required behaviors:
- node pinning to known Proxmox host
- hostpci passthrough for RTX 3090
- required mac_address in network_device
- cloud-init with fixed worker identity

Example fragment:

```hcl
hostpci {
  device = "hostpci0"
  id     = var.gpu_pci_id
  pcie   = true
  rombar = true
}

network_device {
  bridge      = "vmbr0"
  model       = "virtio"
  mac_address = var.mac_address
}
```

### k3s node policy

GPU worker controls:
- label: workload.accelerator/gpu=true
- taint: workload.accelerator/gpu=true:NoSchedule
- only inference workloads tolerate that taint

This prevents general workloads from consuming GPU-node capacity.

## Kubernetes Runtime Design

### GPU software stack

On gpu-worker VM:
- NVIDIA driver compatible with RTX 3090
- NVIDIA container toolkit
- k3s container runtime configured for NVIDIA runtime
- NVIDIA k8s device plugin deployed cluster-wide

Verification gates:
- kubectl describe node shows allocatable nvidia.com/gpu
- test pod requesting nvidia.com/gpu=1 runs successfully
- nvidia-smi returns expected card details inside pod

### Inference provider deployment

Initial provider: Ollama

Deployment constraints:
- nodeSelector matches gpu worker label
- toleration for GPU taint
- resources.limits nvidia.com/gpu: 1
- internal ClusterIP service only

## Gateway and Routing Integration

Gateway is already live and remains the single entry path.

Routing updates:
- Add/update local provider entry for Ollama service DNS in routing config
- Assign local-first behavior for target workload classes
- Preserve existing fallback chains for degraded or exhausted states

No new adapter framework is required. This feature supplies compute capacity; gateway/provider-routing already provides control logic.

## Networking and Security

Controls:
- Ollama service is cluster-internal only
- no direct LAN exposure of raw provider API
- gateway retains auth, normalization, and policy decisions
- optional NetworkPolicy restricts provider service to gateway namespace/pods

## Operations and Reliability

### Failure modes

1. GPU node unavailable (host reboot or VM issue)
- Expected behavior: gateway marks local provider degraded and falls back per routing policy.

2. Driver/plugin failure
- Expected behavior: nvidia.com/gpu disappears; Ollama pod unschedulable; gateway fallback path remains available.

3. Routing misconfiguration
- Expected behavior: startup/health checks fail loudly; no silent provider selection.

### Recovery runbook requirements

Implementation phase must provide:
- host-level VFIO verification commands
- worker-level NVIDIA runtime verification
- k3s plugin health checks
- gateway/provider-routing health checks for local provider availability

## ADRs

### ADR-1: GPU targets a k3s worker VM

Decision: Pass GPU into a dedicated k3s worker, not a standalone inference VM.

Why:
- Directly serves the live gateway running in k3s.
- Keeps inference service discovery and health in-cluster.
- Avoids parallel VM-side inference control plane.

### ADR-2: Static VFIO binding with hard IOMMU gate

Decision: Configure VFIO at host boot and block rollout on unsafe IOMMU grouping.

Why:
- Deterministic boot behavior.
- Prevents fragile runtime rebind behavior.
- Explicitly handles the most common passthrough blocker.

### ADR-3: Required MAC pinning on GPU worker NIC

Decision: mac_address is mandatory in module input.

Why:
- Prevents ARP drift class of incidents seen elsewhere.
- Makes worker identity stable for DNS, firewalling, and runbooks.

### ADR-4: Ollama first, keep gateway routing abstraction

Decision: Initial local provider is Ollama; gateway/provider-routing continues to own routing/fallback.

Why:
- Aligns with existing inference docs and deployed gateway patterns.
- Minimizes new moving parts.

### ADR-5: Dedicated inference-only GPU node policy

Decision: Use taints/labels to isolate GPU node workloads.

Why:
- Protects GPU capacity for intended workloads.
- Reduces noisy-neighbor effects on inference latency.

## Dependencies

- terminus-infra-proxmox: host configuration and passthrough capability
- terminus-inference-gateway: live control plane and API compatibility
- terminus-inference-provider-routing: route policy and fallback behavior

## Open Questions

1. Confirm exact Proxmox node and GPU PCI ID for the RTX 3090.
2. Confirm final static IP/FQDN for gpu-worker-01.
3. Confirm default concurrency and memory policy for 3090-backed Ollama model set.

## Implementation Readiness Notes

Ready for finalizeplan with warnings:
- Requires explicit IOMMU verification output before first apply.
- Requires concrete node/IP/MAC values filled into tfvars.
- Requires gateway routing config update as part of first integrated validation.
