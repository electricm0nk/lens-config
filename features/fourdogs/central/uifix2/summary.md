# Summary: uifix2

**Phase:** dev-ready  
**Goal:** Resolve four UX defects in the fourdogs/central dev environment that degrade operator trust and workflow accuracy: vendor-mismatched catalog tabs, overly broad priority flags, spurious recommendation badges, and a broken "Load Recommendations" button.  
**Status:** draft  
**Updated:** 2026-04-28

## Key Decisions

- Fix scope is develop branch only — no schema changes required
- Defect 4 fix uses risk_score as the primary recommendation qualifier, with suggestedQty as fallback
- No UI redesign — fixes are minimal surgical code changes
- ADR-1: Filter buildCatalogTabs to only tabs with ≥1 classified SKU in source catalog
- ADR-2: Derive prioritySkuIds in FloorWalk.tsx from server query data, not local state
- ADR-3: Gate INCREASED/KAYLEE/DECREASED labels on kayleeQty !== undefined
- ADR-4: Qualify applyKayleeRecommendations items by risk_score OR suggestedQty

## Open Questions

- Does the prod ingest pipeline reliably populate items.o in all environments?
- Should risk_score threshold for auto-recommendation be configurable (default 0)?
- Does risk_score > 0 produce too many items in some catalogs?

## Artifacts Present

- business-plan.md
- tech-plan.md
- sprint-plan.md
- expressplan-adversarial-review.md
- finalizeplan-review.md
- stories/uifix2-1-1.md
- stories/uifix2-1-2.md
- stories/uifix2-1-3.md
- stories/uifix2-1-4.md
