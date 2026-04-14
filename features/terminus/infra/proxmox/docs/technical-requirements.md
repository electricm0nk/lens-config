# Technical Requirements

## Purpose

This document is the lightweight requirements input for the `terminus-infra-proxmox` tech-change initiative.

It replaces a full product PRD because this initiative is infrastructure-only and the selected track is `tech-change`.

## Problem Statement

The Terminus infrastructure needs a Proxmox-focused planning artifact that captures the technical scope, constraints, and success conditions needed to produce an implementation-ready architecture.

## Scope

- Define the Proxmox technical change at the infrastructure layer.
- Capture assumptions, constraints, and interfaces that affect architecture.
- Provide enough structure for TechPlan and later SprintPlan work.

## Non-Goals

- No business-case workflow.
- No market or user-research workflow.
- No feature UX deliverables.

## Required Outcomes

- A concrete architecture for the Proxmox change.
- Clear implementation boundaries and dependencies.
- Explicit operational and security considerations.

## Key Questions To Resolve In TechPlan

- What Proxmox capabilities or resources are being introduced or standardized?
- What systems, services, or automation will integrate with Proxmox?
- What network, identity, secret-management, and storage constraints apply?
- What operational failure modes and recovery expectations must the design address?

## Success Criteria

- The architecture can be implemented without needing a business-planning phase.
- Technical decisions are documented well enough to derive execution work.
- Security, operational, and infrastructure dependencies are explicit.
