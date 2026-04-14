---
type: prd
track: tech-change
status: stub
note: >
  Tech-change track initiatives skip the businessplan phase. This stub satisfies
  the devproposal artifact loader. The product-brief.md serves as the requirements
  document for this initiative.
---

# Product Requirements — terminus-infra-k3s

> **Track: tech-change** — This initiative does not have a traditional PRD.
> Requirements are captured in `product-brief.md` and `architecture.md`.
> See those documents for scope, goals, and success criteria.

## Summary

Deploy a production-grade k3s cluster substrate on Proxmox guest VMs as the runtime platform for all `terminus-platform-*` services.

## Goals

- Reproducible cluster bootstrap from committed artifacts
- Namespace, ingress, TLS, and secrets infrastructure available to downstream features
- Explicit consumer contract so downstream features can declare dependencies

## Out of Scope

- Vault deployment (`terminus-infra-secrets`)
- Prometheus monitoring stack (separate initiative)
- Application workloads (`terminus-platform-*`)

## Reference Documents

- `product-brief.md` — full project brief and problem statement
- `architecture.md` — all technical decisions, toolchain, topology, patterns, structure
