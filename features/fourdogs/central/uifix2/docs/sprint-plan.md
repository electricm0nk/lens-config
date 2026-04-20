---
feature: uifix2
doc_type: sprint-plan
status: draft
updated_at: "2026-04-28"
---

# Sprint Plan — uifix2

**Sprint:** uifix2-1 (single milestone)  
**Target branch:** `develop` (fourdogs-central-ui)  
**Track:** express  
**Sizing:** 4 stories × ~2 points each = ~8 points

---

## Sprint Goal

Deliver all four UI defect fixes to the fourdogs/central `develop` environment,
restoring operator trust in the ordering worksheet's brand tabs, priority flags,
tier labels, and Kaylee recommendation button.

---

## Stories

| ID | Title | File | Points | Repo |
|---|---|---|---|---|
| uifix2-1-1 | Fix brand tab filtering in catalogTabs | stories/uifix2-1-1.md | 2 | fourdogs-central-ui |
| uifix2-1-2 | Fix floor walk priority badge source | stories/uifix2-1-2.md | 2 | fourdogs-central-ui |
| uifix2-1-3 | Gate tier labels on Kaylee recommendation presence | stories/uifix2-1-3.md | 2 | fourdogs-central-ui |
| uifix2-1-4 | Fix Load Recommendations button qualification | stories/uifix2-1-4.md | 2 | fourdogs-central-ui |

---

## Dependencies

- velocitysignalwiring2 is live — no ordering constraint between stories
- Stories are independent and can be implemented in parallel

---

## Acceptance Gate

All four stories done → manual smoke-test of 4 acceptance criteria in dev → PR to `develop`.
