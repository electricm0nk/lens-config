---
title: "terminus-inference-gateway — Product Brief"
initiative: terminus-inference-gateway
track: feature
phase: preplan
status: approved
created: 2026-04-02
source: synthesized from terminus-inference-service-plan.md
stepsCompleted: []
---

# Product Brief: terminus-inference-gateway

## Problem Statement

The terminus platform hosts multiple AI-powered workflows (e.g. DailyBriefingWorkflow) that invoke inference directly. As local runtime, cloud fallback, and eventual GPU acceleration enter the picture, each consumer would need to know which backend to call, how to handle failover, and how to manage auth and rate constraints. There is no single stable surface today — adding a provider, changing a model host, or shifting capacity requires touching every consumer.

## Opportunity

Establish a single OpenAI-compatible API gateway in front of all inference runtimes. Clients bind to one stable contract. Provider selection, initialization, normalization, auth enforcement, and rate guardrails are handled once at the gateway boundary. When a new provider is added or an existing one is replaced, zero client code changes are required.

## Target Users

| User | Context |
|------|---------|
| Platform services (terminus.platform) | Call inference via a stable endpoint in workflows and background jobs |
| Batch orchestrator (future) | Submits long-running batch jobs through the gateway route |
| Developer (local) | Runs inference locally for development without managing provider SDKs directly |

## Proposed Solution

A lightweight API gateway service that:
- Exposes an OpenAI-compatible endpoint surface (chat completions, completions)
- Normalizes request and response contracts across any connected provider
- Enforces auth and rate guardrails at the gateway boundary
- Delegates provider-specific routing policy to a separate feature (terminus-inference-provider-routing)
- Is backed initially by local CPU runtime (terminus-inference-runtime-cpu) and later by GPU and cloud providers

## Scope

**In scope for this feature:**
- OpenAI-compatible API endpoint surface (`/v1/chat/completions`, etc.)
- Request and response contract normalization across providers
- Auth enforcement (API key or token validation at boundary)
- Provider adapter interface — pluggable, not provider-specific in the gateway itself

**Out of scope for this feature:**
- Provider-specific routing policy or route profiles (terminus-inference-provider-routing)
- Local runtime setup or model loading (terminus-inference-runtime-cpu)
- Batch orchestration, checkpointing, unattended execution (terminus-inference-job-orchestrator)
- GPU runtime (terminus-inference-runtime-gpu)

## Acceptance Targets

1. A client configured for Provider A can switch to Provider B by changing config only — no code change
2. Auth and rate guardrails are enforced at the gateway boundary before any provider call
3. The gateway returns normalized responses regardless of which provider backend is active
4. Contract test suite passes with a stable schema for all exposed routes

## Dependencies

| Dependency | Why |
|-----------|-----|
| terminus-infra-secrets | Gateway reads API keys and provider credentials from Vault at runtime |
| terminus-infra-k3s | Gateway runs as a k3s workload on the terminus substrate |

## Non-Negotiable Constraints (from service plan)

1. API contract stability over model quality experimentation — the contract must not change to chase provider features
2. Config-driven routing over hardcoded provider branches — no provider name in gateway handler logic
3. No routing policy logic embedded in gateway handlers — that belongs to provider-routing feature

## Success Signal

A consuming service (e.g. DailyBriefingWorkflow worker) can point its LLM client at the gateway URL, authenticate, and receive normalized completions — without knowing or caring which provider is currently active.
