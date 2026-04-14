---
verdict: PASS_WITH_NOTES
initiative: terminus-platform-temporal
audience: medium
entry_gate: adversarial-review
mode: party
lead: john
participants: [john, mary, bob]
artifact_reviewed: docs/terminus/platform/temporal/architecture.md
reviewed_at: '2026-03-31'
---

# Adversarial Review Report — terminus-platform-temporal Architecture

**Focus:** Meets spec? Practical? Sprintable?

**Verdict: PASS_WITH_NOTES**

The architecture is fundamentally sound. Tech stack is correct and well-chosen. Patterns are comprehensive. Project structure is fully defined to filename level. The initiative is approved to proceed to sprintplan with the notes and required resolutions documented below.

---

## Review Panel

| Persona | Role | Focus Area |
|---|---|---|
| John (PM) — lead | Product Manager | Meets spec? Requirements gaps? |
| Mary (Analyst) | Business Analyst | Practical? Feasible? Real-world risks? |
| Bob (Scrum Master) | Scrum Master | Sprintable? Story-ready? |

---

## Blockers (Must Resolve Before Story Writing)

### B1 — ESO ClusterSecretStore Name Not Specified

The architecture explicitly flags this as "resolve at story time" but `ExternalSecret` manifests cannot be written without the ClusterSecretStore name. Multiple stories (ESO ExternalSecrets, k8s namespace setup) depend on this. Without anchoring it now, stories will diverge or stall.

**Resolution:** Add `eso_cluster_secret_store_name` to the architecture document or initiative config. Verify from domain architecture (expected to be the existing store serving other terminus workloads).

---

### B2 — ArgoCD Application Manifest Location Not Documented

The architecture says "follow the existing App-of-Apps pattern" but does not specify the repo, directory, or file naming convention. An implementation agent tackling the Temporal server story will have insufficient context to place the `Application` CR correctly.

**Resolution:** Document the pattern explicitly — e.g., "ArgoCD Application CR lives in `{repo}/{path}/` — follow the pattern from the crossplane initiative."

---

### B3 — "Temporal Server" and "Temporal Worker" Stories Not Decomposed

Stories 3 and 4 in the implementation handoff are monolithic. "Temporal server story" alone encompasses: k8s namespace, ESO ExternalSecrets, Helm values file, ArgoCD Application, Traefik IngressRoute, cert-manager Certificate. These are 4–6 independently deployable stories.

**Resolution:** Decompose during sprintplan into atomic stories. Each story should be independently deployable, test-verifiable, and have a clear definition of done. Suggested decomposition:
- Story: k8s namespace + RBAC
- Story: ESO ExternalSecrets (DB + visibility)
- Story: Temporal server Helm values + ArgoCD Application
- Story: Traefik IngressRoute + cert-manager
- Story: Worker Helm chart + ArgoCD Application
- Story: TypeScript worker scaffold + CI image build

---

## Important Notes (Address During Story Writing)

### N1 — Worker Health Probe Undefined

"exec against Temporal connection" is too vague to implement. The `@temporalio/sdk` does not expose an HTTP health endpoint. The implementing agent needs a specific command — likely a script that attempts a `grpc` call to `temporal-frontend:7233` or checks the worker's internal state.

**Action:** Define the health probe command in the worker story acceptance criteria, or add it to the architecture document.

---

### N2 — No Smoke Test / Acceptance Criteria for "Temporal Operational"

The architecture defines how to build and deploy Temporal but not how to verify it works as a system. After all deployment stories land, there should be a verifiable end state.

**Action:** Add a verification story or acceptance criteria to the final worker story: e.g., "trigger a test workflow via `tctl` or Web UI and verify completion in the visibility store."

---

### N3 — DB Provisioning Story Has No Implementation Approach

"Patroni admin task — separate story, not in this repo" does not give implementing agents enough to work with. Is this a SQL script? A k8s Job? A Helm hook?

**Action:** Define the mechanism in the Story Zero B description. Suggested: SQL script committed to a docs/scripts location, run manually against Patroni primary with documented connection steps.

---

### N4 — `terminus-agent-dailybriefing` Dependency Not Sequenced

`terminus-agent-dailybriefing` is already in `large-sprintplan` and is the primary Temporal consumer. If dailybriefing dev begins before Temporal is operational, integration will be impossible.

**Action:** Story planning must ensure Temporal server + worker stories land before dailybriefing dev stories that depend on Temporal. Document this dependency in sprint planning.

---

### N5 — Worker Restart Behavior During Deploy Unaddressed

With `replicas: 1`, a worker restart drops any in-flight activity. Temporal will retry per retry policy, but acceptable latency during a rolling deploy is not documented.

**Action:** Add a note in the worker story or values.yaml that acknowledges this behavior. For homelab use, this is acceptable but should be explicit.

---

### N6 — Definition of Done Missing for Story Zero A (Monorepo Scaffold)

"Initialize monorepo scaffold" is unbounded without DoD. An agent could stop at any point and claim the story is done.

**Action:** Story Zero A DoD must include: `npm install` succeeds, `npm run build` (if defined) succeeds, `shared/types` TypeScript compiles clean, CI workflow file exists and is syntactically valid.

---

### N7 — Worker Image Bootstrap Gap

The CI workflow builds and pushes the worker image on merge to the default branch. But on first deploy, no image exists yet. The worker Deployment will fail until an image is available.

**Action:** Define the bootstrap sequence in the worker story: either a manual first-build step, a bootstrap workflow trigger, or a placeholder image pattern. This should not be discovered at deploy time.

---

### N8 — `k8s/` Directory Mixes Story Boundaries

`namespace.yaml`, `external-secret-db.yaml`, `external-secret-visibility-db.yaml`, and `rbac-worker.yaml` are logically separate deployable units. A single story covering all of `k8s/` is too large.

**Action:** Per the decomposition in B3, namespace/RBAC should be one story, ESO ExternalSecrets another. The architecture structure is correct — but story tickets must not bundle these.

---

## Acknowledged Observations (No Action Required)

| # | Observation | Disposition |
|---|---|---|
| A1 | Helm chart `1.0.0-rc.3` is a release candidate | Acceptable for homelab; monitor for `1.0.0` GA release |
| A2 | Patroni IP `10.0.0.56` is hard-coded with no fallback | Intentional; document in values.yaml comment |
| A3 | `services/grafana/` and `services/prometheus/` stubs pre-empt future initiatives | Remove stubs from Story Zero A scope; let those initiatives define their own structure |

---

## Panel Verdict Summary

The architecture is approved to advance. Blockers B1, B2, and B3 must be resolved during sprintplan story writing — they are not architectural defects but rather documentation gaps that will cause story-level ambiguity. Notes N1–N8 should be incorporated as story acceptance criteria or sprint planning constraints.

**Architecture verdict: PASS_WITH_NOTES — proceed to sprintplan.**
