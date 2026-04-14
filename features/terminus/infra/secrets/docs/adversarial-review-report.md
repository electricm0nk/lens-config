---
verdict: PASS_WITH_NOTES
initiative: terminus-infra-secrets
phase: devproposal-entry-gate
audience: medium
reviewed_at: 2026-03-24T01:10:00Z
reviewed_by: [john, winston, mary, sally, bob]
artifacts_reviewed:
  - docs/terminus/infra/secrets/prd.md
  - docs/terminus/infra/secrets/architecture.md
mode: party
lifecycle_gate: audience-promotion-small-to-medium
---

# Adversarial Review Report — terminus-infra-secrets

**Gate:** Entry gate for medium audience (small → medium promotion)
**Mode:** Party — all review boards, realistic challenge
**Verdict:** PASS_WITH_NOTES — DevProposal may proceed; notes must appear as stories

---

## Board 1 — PRD Review

**Lead:** Winston (Architect)
**Board:** Winston (Architect), Mary (Analyst), Sally (UX Designer)
**Focus:** Buildable? Well-researched? Operator-journey aligned?

### Winston — Buildability

PRD is buildable. Toolchain is fully locked. Two-layer model (SOPS+age authoring / Vault
KV v2 runtime) is clean. Bootstrap problem identified as a first-class architectural
challenge — exactly correct framing. DR as a design constraint (not scope item) is a
strong insight; blast-and-repave and snapshot restore as two distinct paths is the right
bifurcation.

**Concern — FR24 automated rotation Growth direction:** Scoped to Growth with no sketch
of the automation mechanism. Manual rotation in MVP is correct, but without Growth-phase
direction an implementation team could build the manual path in a way that blocks
automation later. DevProposal should include a story that designs the automated rotation
stub — creates the hook, not the automation.

**Board 1 / Winston verdict:** PASS

### Mary — Research Depth

Research depth is appropriate for a solo-operator homelab initiative with professional
skills transfer intent. Raft vs. Consul storage decision is well-reasoned. SoD deviation
has appropriate compensating controls. Monitoring-readiness as a design constraint (not
afterthought) shows system-level thinking.

**Concern — AppRole token TTL vs secret TTL:** No mention of secret lease expiry behavior
for AppRole tokens. Vault AppRole tokens have their own TTL separate from the underlying
secret TTL. What happens when an AppRole *token* expires while the workload is running?
Not a blocking PRD gap — implementation-time decision — but must surface in stories.

**Board 1 / Mary verdict:** PASS

### Sally — Operator Journey Alignment

Operator journeys are well-specified. Journey 1 (Day-0 Bootstrap) is the primary journey
with the right scope anchor: `tofu apply` → operational Vault. The "returning operator
after 6 months" constraint (NFR14) forces every procedure to be documentation-complete.
Journey 5a/5b (blast-and-repave + snapshot restore) as distinct journeys is correct.
Journey 4 (Break-Glass) has a clear mental model.

**Concern — New workload onboarding experience:** No operator-journey story for
*onboarding a new workload for the first time*. The AppRole onboarding checklist exists
in the architecture but the *end-to-end operator verification* of the journey needs a
dedicated story with acceptance criteria.

**Board 1 / Sally verdict:** PASS

### Board 1 Verdict: PASS

---

## Board 2 — Architecture Review

**Lead:** John (PM)
**Board:** John (PM), Mary (Analyst), Bob (Scrum Master)
**Focus:** Meets spec? Practical? Sprintable?

### John — Spec Coverage

Architecture coverage against 39 FRs + 23 NFRs is complete. Requirements-to-structure
mapping table is comprehensive. NFR9 (consumer graceful degradation) correctly delegated
to consumers with documentation obligation. FR39 (CVE automation) correctly delegated to
`terminus-infra-dependency-health` future initiative.

**Concern 1 — Vault provider alias:** Flagged as HIGH in gap analysis but buried in a
footnote. This is a hidden complexity that can derail the first sprint. Must have its own
dedicated story before any `vault-config` work begins.

**Concern 2 — TLS CA cert distribution:** Architecture mandates TLS on Vault listener
(NFR2) and commits CA cert to git. The client trust configuration (where Ansible sets
`VAULT_CACERT`, where OpenTofu uses `ca_cert_pem` in the Vault provider config) is not
explicitly detailed. DevProposal story needed.

**Board 2 / John verdict:** PASS_WITH_NOTES

### Mary — Practicality

Architecture is practical for solo homelab. Module layering is clean. Decision log
(D1-D8) gives implementers the *why*. No unnecessary external dependencies.

**Concern 1 — Vault provider alias at init time (HIGH):** The `tofu apply` sequence
creates a circular dependency risk point. `vault-core` uses the Vault provider configured
with the root token from `vault operator init`. If OpenTofu initializes the Vault provider
before the null_resource provisioner runs vault init, the apply fails with a provider auth
error. The architecture notes "provider alias required" but only in a footnote. A junior
implementer could burn a day here. Must become a dedicated story with explicit guidance.

**Concern 2 — SOPS state encryption discipline:** D4 specifies manual discipline via a
Makefile target. No mention of a git `pre-commit` hook to *prevent* accidental unencrypted
state commits. Adding a hook alongside the Makefile target eliminates the human failure
mode. Should be a story.

**Board 2 / Mary verdict:** PASS_WITH_NOTES

### Bob — Sprintability

Module structure maps directly onto natural epics. Day-0 vs. Day-2 artifact separation
creates a clean epic boundary. Directory tree is specific enough to write stories
referencing actual file names.

**Concern 1 — Verification suite (D8) not scoped as first-class:** Architecture calls it
a "first-class MVP deliverable" but without explicit story weighting it will be deferred
to the end as cleanup work. Must be a dedicated epic or story *gated by* other
implementation stories.

**Concern 2 — SOPS state encryption as a process story:** Needs acceptance criteria:
"Makefile target exists, pre-commit hook exists, `git add terraform.tfstate` without
prior encryption is blocked by hook."

**Concern 3 — Break-glass runbook must be exercised:** NFR14 mandates runbooks executable
without prior context. Stories for break-glass and blast-and-repave must include
acceptance criteria requiring at least one exercised run, not just documentation.

**Concern 4 — Three workload identities are all required:** Success criteria specify ≥3
distinct workload identities (opentofu, k3s, openclaw). All three must be in scope, not
just one as a proof of concept.

**Board 2 / Bob verdict:** PASS_WITH_NOTES

### Board 2 Verdict: PASS_WITH_NOTES

---

## Consolidated DevProposal Story Requirements

The following items **must appear as stories** in the DevProposal to proceed past this gate:

| # | Story Requirement | Source | Priority |
|---|---|---|---|
| S1 | Vault provider alias pattern for root token handoff — dedicated story with explicit implementation guidance | John, Mary (arch concerns) | HIGH |
| S2 | TLS CA cert distribution to Ansible and OpenTofu clients | John | HIGH |
| S3 | SOPS state encryption guard — Makefile target + pre-commit hook that blocks unencrypted state commit | Mary, Bob | HIGH |
| S4 | Isolation verification suite (D8) as first-class MVP story, gated by other implementation | Bob | HIGH |
| S5 | Break-glass runbook — must include acceptance criteria requiring one exercised run | Bob | MEDIUM |
| S6 | Blast-and-repave procedure — acceptance criteria requires one full exercised reprovisioning | Bob | MEDIUM |
| S7 | New workload onboarding end-to-end — operator journey verification story (all three: opentofu, k3s, openclaw) | Sally | MEDIUM |
| S8 | AppRole token TTL policy — document and configure token TTL strategy per workload identity | Mary | MEDIUM |
| S9 | Automated rotation stub/hook design — Growth phase direction sketched in MVP even if not implemented | Winston | LOW |

---

## Overall Verdict

**PASS_WITH_NOTES**

All planning artifacts (PRD + Architecture) are of sufficient quality to proceed to
DevProposal. The review notes represent implementation complexity that must be explicitly
captured as stories, not left to implementation-time discovery. No rework of PRD or
Architecture is required.

DevProposal execution may begin.
