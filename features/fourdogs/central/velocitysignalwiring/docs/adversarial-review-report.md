---
verdict: PASS_WITH_NOTES
initiative: fourdogs-central-velocitysignalwiring
phase: small→medium entry gate
artifact_reviewed: architecture
review_date: '2026-04-14'
panel: [pm, analyst, sm]
---

# Adversarial Review — fourdogs-central-velocitysignalwiring

**Gate:** small → medium entry gate (adversarial-review, party mode)
**Artifact:** `docs/fourdogs/central/velocitysignalwiring/architecture.md`
**Focus:** Meets spec? Practical? Sprintable?
**Panel:** Lord Commander Creed (PM) · Inquisitor Greyfax (Analyst) · High Chaplain Grimaldus (Scrum Master)

---

## Verdict: PASS_WITH_NOTES

All findings are informational. No hard-gate blockers. Promotion to medium audience is approved.

---

## Findings

| ID | Reviewer | Finding | Gate Level |
|----|----------|---------|------------|
| R-1 | Lord Commander Creed | AD-2 ownership ambiguity: "runs as a post-cycle step inside RunCycle" implies the classifier lives in the emailfetcher process, but the component map places `velocity.go` inside the central API. A developer reading AD-2 in isolation will implement the wrong process boundary. Component map resolves it — ADR text does not. | Informational |
| R-2 | Inquisitor Greyfax | Architecture internally inconsistent on the key structural decision: AD-2 says "inside RunCycle" (emailfetcher process); component map says `internal/handler/velocity.go` (central API process). These are different runtime processes. The correct architecture (HTTP trigger from emailfetcher to central API) is only determinable from the component map, not from the decision text. | Informational |
| R-3 | Inquisitor Greyfax | Risk register marks "EtailPet trigger API credentials/endpoint unknown" as High likelihood with no resolution owner or release gate. Recommend a backlog item be named explicitly before the EtailPet risk becomes a silent non-delivery. | Informational |
| R-4 | High Chaplain Grimaldus | Six implementation units are clearly scoped and sprint-sized. Serial constraint (sqlc regen follows migration) noted and manageable. However, unit 5 (emailfetcher post-cycle trigger) will be story-written incorrectly if AD-2 is not corrected before devproposal. The story must describe an HTTP outbound call, not an embedded function call. | Informational — fix before devproposal |

---

## Strengths

- **Data-backed thresholds:** 28,067 rows queried, threshold sensitivity table produced before committing to fast≥2.0/medium≥0.5 upw. No guesswork.
- **MAX anchor decision (AD-3):** Correctly reasoned. Anchoring to `MAX(sale_date)` rather than `CURRENT_DATE` prevents silent window shrinkage during manual import cadence. This is the kind of decision that prevents a silent bug six months from now.
- **EtailPet stub (AD-5):** Correct doctrine. Declares intent, ships no-op, unblocks delivery without waiting for vendor API credentials.
- **Handler already implemented:** `internal/handler/velocity.go` with tests passing is ahead of the architecture gate, which is the correct order of operations for a tech-change track.
- **Null → slow semantics (AD-4):** Correct. Items with no sales in 60-day window should not show as HOT. The frontend fallback is semantically accurate.

---

## Pre-DevProposal Action (Required)

> **Amend AD-2** in `docs/fourdogs/central/velocitysignalwiring/architecture.md` to clarify:
> - The velocity classifier runs in the **central API process** (`cmd/api`)
> - The emailfetcher triggers it via **HTTP: `POST /internal/velocity/refresh`**
> - The ADR decision text should read: "The emailfetcher, upon detecting `salesRowsImported > 0`, calls `POST /internal/velocity/refresh` on the central API. The central API executes the UPDATE SQL."
> - Remove or qualify the phrase "runs as a post-cycle step inside RunCycle" which implies in-process execution

This correction must precede story drafting in devproposal to prevent the emailfetcher story from implementing the wrong process boundary.

---

## Promotion Decision

**APPROVED — small → medium**

The architecture is sound, data-backed, and sprintable. The identified inconsistency (AD-2 text vs component map) is informational and must be resolved before devproposal story drafting, but does not block promotion.
