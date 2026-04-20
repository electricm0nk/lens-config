---
feature: transactions
doc_type: implementation-readiness
status: draft
goal: "Assess readiness to begin EPIC-1 implementation; surface all remaining risks and prerequisites."
key_decisions:
  - Feature is NOT ready to start EPIC-1 — EPIC-0 (spike) must complete first
  - All platform prerequisites are in place; code can begin immediately after EPIC-0 closes
  - No new platform services, repos, or team members required
open_questions:
  - eTailPet API auth mechanism (blocks EPIC-1 start)
  - eTailPet trigger endpoint URL and HTTP method (blocks EPIC-1 start)
  - eTailPet rate limit (informs schedule cadence validation in EPIC-2)
depends_on: []
blocks: []
updated_at: "2026-04-20T00:00:00Z"
---

# Implementation Readiness: transactions

## Summary

| Area | Status | Notes |
|---|---|---|
| Planning artifacts complete | ✅ READY | business-plan, tech-plan, sprint-plan, 8 stories, epics, finalizeplan-review all committed |
| Architecture approved | ✅ READY | emailfetcher pattern; 4 ADRs documented and reviewed |
| Platform prerequisites | ✅ READY | fourdogs-central repo, GHCR, ArgoCD `fourdogs` project, ESO, Vault all exist |
| eTailPet API open questions | ⛔ BLOCKING | Must resolve before EPIC-1 coding begins |
| Implementation team | ✅ READY | Solo (Todd); no coordination required |
| Sprint 0 spike | ⛔ REQUIRED FIRST | EPIC-0 is the gate; Sprint 1 does not start until Done |

**Overall verdict: READY TO START EPIC-0 (spike). EPIC-1 blocked until spike is complete.**

---

## Platform Prerequisites

| Prerequisite | Status | Verified By |
|---|---|---|
| `fourdogs-central` GitHub repo exists | ✅ | Existing repo |
| GHCR image registry wired for fourdogs | ✅ | emailfetcher images already published there |
| ArgoCD `fourdogs` project exists | ✅ | Other fourdogs services deployed to it |
| `ESO ClusterSecretStore/vault-backend` exists | ✅ | Used by all fourdogs services |
| Vault `secret/terminus/fourdogs/` path accessible | ✅ | Used by fourdogs-central and emailfetcher |
| Go module in `fourdogs-central` | ✅ | Existing module; new packages extend it |
| k3s cluster available (dev) | ✅ | All fourdogs services deployed there |

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| eTailPet API undocumented or support unresponsive | Medium | High — blocks EPIC-1 start | EPIC-0 has a 5-day time-box; escalate to Todd on day 3 if no resolution path |
| eTailPet API uses OAuth (more complex auth) | Low | Medium — adds ~1 sprint point | ADR-2 notes OAuth path; sprint estimate buffer included |
| emailfetcher inbox unavailable during staging validation | Low | Medium — blocks EPIC-2 sign-off | Ensure emailfetcher healthy before starting EPIC-2 staging validation |
| Vault secret path conflicts with existing entries | Low | Low — naming is unique | Path `secret/terminus/fourdogs/etailpet-api` is a new, unique path |
| `trigger_success` logged but eTailPet silently drops export | Low | Medium — v1 has no delivery confirmation | Known gap; compensating signal is emailfetcher inbox monitoring |
| Secret rotation blocks trigger after Vault rotation | Low | Low — known limitation | ADR-4 documents rotation procedure; operator handles manually in v1 |

---

## Implementation Sequence

```
EPIC-0 (spike)          ← start here; 5-day time-box
    ↓
EPIC-1 (implementation) ← Go code, unit tests, CI; ~2 weeks
    ↓
EPIC-2 (deploy+validate) ← Helm, ESO, staging, runbook; ~1 week
    ↓
Production enable (TRIGGER_ENABLED=true) → monitor 7 days → close feature
```

---

## Definition of Ready for EPIC-1

The following must be true before the first EPIC-1 story begins:

- [ ] `etailpet-api-spike.md` committed with all 5 open questions answered
- [ ] Vault secret at `secret/terminus/fourdogs/etailpet-api` provisioned and verified (`vault kv get secret/terminus/fourdogs/etailpet-api`)
- [ ] `ETAILPET_TRIGGER_URL` confirmed and recorded in spike document
- [ ] Auth mechanism confirmed and recorded; client auth pattern selected (header/OAuth/basic)
- [ ] Rate limit documented (informs whether daily cadence is safe)
