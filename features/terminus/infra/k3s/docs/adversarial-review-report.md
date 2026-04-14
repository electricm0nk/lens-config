---
type: adversarial-review
initiative: terminus-infra-k3s
phase: small→medium promotion
entry_gate: adversarial-review
mode: party
verdict: PASS_WITH_NOTES
reviewedAt: '2026-03-28'
artifacts_reviewed:
  - docs/terminus/infra/k3s/product-brief.md
  - docs/terminus/infra/k3s/prd.md (stub — tech-change track)
  - docs/terminus/infra/k3s/architecture.md
reviewers:
  product-brief: [john, winston, sally]
  architecture: [john, mary, bob]
---

# Adversarial Review Report — terminus-infra-k3s

**Verdict: PASS_WITH_NOTES**

Architecture is sound and devproposal may proceed. Four story-blocking gaps must be resolved within story definitions before sprint planning. Six advisory improvements are noted.

---

## Product Brief Review (John · Winston · Sally)

### Findings

1. **No measurable success criteria** — "stable runtime contract" and "dependable substrate" are unmeasurable. Success criteria must include observable, testable conditions (e.g. all nodes Ready, specific namespaces present, downstream feature can deploy without cluster-level changes).

2. **k3s not justified over alternatives** — the brief describes what the cluster must do but never explains why k3s specifically. Any lightweight Kubernetes distribution satisfies the stated requirements. A one-line justification (simplicity, embedded etcd, single-binary, homelab fit) should appear in the brief.

3. **User journey is incomplete** — the journey ends at "operator provisions VMs." No journey through: cluster bootstrap, first downstream feature onboarding, upgrade procedure, node failure recovery. The journey as written validates only the provisioning phase.

4. **Single-user brief in a multi-consumer system** — "Todd" is the only defined user. Downstream feature owners (even if the same person) are a distinct persona. The brief doesn't capture the consumer perspective: a developer of `terminus-infra-postgres` who needs to know what the cluster provides.

5. **Key Differentiators are process claims, not differentiators** — "Cluster-as-product mindset" and "Security-first foundation" are internal process commitments, not differentiators vs. the alternatives described (one-off installs, application-first design). Reframe as outcomes.

---

## Architecture Review (John · Mary · Bob)

### Story-Blocking Findings (must resolve in stories before sprint planning)

**SB-1: Worker node count not decided**
The architecture specifies 3 control-plane nodes but worker node count is absent. Epic and story scoping requires knowing the target topology. Minimum: decide (e.g. 2 workers) and document in architecture.md or in the first story.

**SB-2: Namespace list never enumerated**
Consumer contract (Step 4) states "namespaces exist per the namespace model (defined in Step 6)." Step 6 only shows a filename (`namespaces.yaml`) without listing actual namespace names. Downstream stories that reference a specific namespace cannot be validated without this list. Must be resolved as a story prerequisite.

**SB-3: Component versions not pinned**
All toolchain components have versions (opentofu-1.11.0, ansible-core-2.19, vault-kv-v2-1.21.4) but in-cluster components (ArgoCD, cert-manager, MetalLB, External Secrets Operator, Traefik) have no versions. Every story implementing these will spawn a version debate. Pin versions in architecture.md or in a dedicated `versions.yaml` before implementation begins.

**SB-4: ESO → Vault authentication method not specified**
The architecture decides to use External Secrets Operator with Vault KV v2 but does not specify how ESO authenticates to Vault (Kubernetes auth via ServiceAccount JWT? AppRole? Token). This is a story-blocking gap — the story "implement ESO" cannot be written without it. Kubernetes auth method is the recommended default for in-cluster operators.

### Advisory Findings (important but not sprint-blocking)

**A-1: k3s HA bootstrap sequence not documented**
3-node HA requires `--cluster-init` on the first server, then `--server https://<first-node>:6443` for the second and third. This Ansible role-level detail is absent from architecture.md. It won't block devproposal but will block the first bootstrap story.

**A-2: Bootstrap chicken-and-egg for ESO**
ESO needs Vault credentials to start syncing. How does ESO's initial Vault token get into the cluster? Ansible `kubectl create secret` at bootstrap time (before ArgoCD takes over) is the expected pattern but is not documented. Add a bootstrap note to the architecture.

**A-3: MetalLB IP pool DHCP overlap risk**
The pool `10.0.0.126–10.0.0.254` must be confirmed outside DHCP scope. If the router's DHCP range overlaps, LoadBalancer VIPs will cause IP conflicts. Confirm and document the DHCP boundary.

**A-4: ArgoCD self-management recovery**
`selfHeal: true` on all apps including ArgoCD itself. The fallback for when ArgoCD is unavailable (manual `kubectl apply -k` or `helm install`) is not documented. Add to runbook-bootstrap.md.

**A-5: No phased delivery priority**
The architecture identifies 8+ substrate components but provides no sequencing priority. Devproposal story authoring will need to establish this order. Recommended sequence: (1) k3s bootstrap + HA, (2) namespaces + RBAC, (3) ArgoCD bootstrap, (4) MetalLB, (5) Traefik, (6) cert-manager, (7) ESO, (8) consumer contract validation.

**A-6: kubeconfig distribution not addressed**
"Direct kubeconfig" access is decided but not how the operator retrieves, stores securely, or rotates the admin kubeconfig after bootstrap. This is an operational gap for recovery scenarios.

---

## Verdict Rationale

**PASS_WITH_NOTES** — The architecture demonstrates clear scope, sound technology decisions, an explicit consumer contract, and comprehensive patterns. The four story-blocking gaps (SB-1 through SB-4) are well-scoped and resolvable within story definitions without requiring architecture rework. The advisory findings are engineering-quality improvements that strengthen operational readiness.

Devproposal may proceed. The four story-blocking findings must appear as explicit acceptance criteria or prerequisites in the relevant stories.
