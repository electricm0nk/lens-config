---
feature: uifix2
doc_type: tech-plan
status: draft
goal: >
  Four surgical code changes in fourdogs-central-ui to fix vendor-tab scoping,
  floor-walk priority source, tier-label gating, and recommendation-button qualification.
key_decisions:
  - ADR-1: Filter buildCatalogTabs to only tabs with ≥1 classified SKU in source catalog
  - ADR-2: Derive prioritySkuIds in FloorWalk.tsx from server query data, not local state
  - ADR-3: Gate INCREASED/KAYLEE/DECREASED labels on kayleeQty !== undefined
  - ADR-4: Qualify applyKayleeRecommendations items by risk_score OR suggestedQty (not suggestedQty alone)
open_questions:
  - Does risk_score > 0 produce too many items in some catalogs (> 200)? If so, consider risk_score >= 20 threshold.
depends_on:
  - velocitysignalwiring2 (live)
blocks: []
updated_at: "2026-04-28"
---

# Tech Plan — uifix2

## Technical Summary

All four defects are **frontend-only** changes in `fourdogs-central-ui`. No backend
schema migrations, no API contract changes, no Kaylee model changes required.

The backend `GET /v1/items` endpoint already returns all necessary data:
`i.o` (suggestedQty), `i.dos_days`, `i.risk_score`, `i.vc`, `i.t` (tab), `i.b`
(brand), and all other fields. The normalization layer in `use_vendor_catalog.ts`
already maps most of these. One mapping (`riskScore`) should be verified.

Effort estimate: 4 focused story-sized changes, each touching one logical concern.

---

## Architecture Overview

```
GET /v1/items?vendor_id=X
        │
        ▼
use_vendor_catalog.ts   normalizeItem()
        │  row.o → suggestedQty
        │  row.dos_days → dosDays
        │  row.risk_score → riskScore
        │  row.vc → velocity (velocity class)
        │  row.t → tab
        │
        ▼
ChairSku[]   (sourceSkus)
        │
        ├──► catalogTabs.ts  buildCatalogTabs(sourceSkus)   ← Defect 1
        │
        ├──► FloorWalk.tsx   prioritySkuIds                  ← Defect 2
        │       from: lineItems  →  from: floorWalkLinesQuery.data
        │
        └──► OrderDetail.tsx
                ├── resolveSkuTier(qty, kayleeQty, isImported)  ← Defect 3
                └── applyKayleeRecommendations()                ← Defect 4
```

---

## Design Decisions

### ADR-1 — Filter `buildCatalogTabs` to vendor-present tabs only

**Context:**
`buildCatalogTabs(skus)` in `catalogTabs.ts` always appends `STATIC_REST_TABS`:
`['wellness','leashes','beds-apparel','toys','ruffwear','tractor-supply','reedy-fork','everything-else']`.
When ordering from a vendor that doesn't carry Ruffwear, Tractor Supply, or Reedy Fork
items, those tabs appear with 0 items — confusing and misleading.

**Decision:**
Change `buildCatalogTabs` to compute the set of tabs by classifying each SKU in
`sourceSkus` and only returning tabs that have at least one SKU classified into them.
`STATIC_REST_TABS` becomes the **ordered template** for tab sorting, not an
unconditional inclusion list.

**Implementation (conceptual diff):**
```typescript
// Before
export function buildCatalogTabs(skus: ChairSku[]): CatalogTab[] {
  const tabSet = new Set<string>([...STATIC_REST_TABS]);
  skus.forEach(s => tabSet.add(classifyCatalogTab(s)));
  return [...tabSet].map(tabId => ({ id: tabId, label: TAB_LABELS[tabId] ?? tabId }));
}

// After
export function buildCatalogTabs(skus: ChairSku[]): CatalogTab[] {
  const occupied = new Set(skus.map(s => classifyCatalogTab(s)));
  // Preserve STATIC_REST_TABS order for tabs that do appear; append any others
  const ordered = STATIC_REST_TABS.filter(t => occupied.has(t));
  occupied.forEach(t => { if (!ordered.includes(t)) ordered.push(t); });
  return ordered.map(tabId => ({ id: tabId, label: TAB_LABELS[tabId] ?? tabId }));
}
```

**Consequences:**
- Positive: tabs are always vendor-scoped; no phantom brand tabs
- Neutral: tab list may be shorter for vendors with narrow catalogs
- Risk: if `skus` is empty during loading, all tabs temporarily disappear → guard with
  loading state (tabs only rendered after catalog load completes)

---

### ADR-2 — Derive `prioritySkuIds` in `FloorWalk.tsx` from server data

**Context:**
`FloorWalk.tsx` computes:
```typescript
const prioritySkuIds = useMemo(
  () => new Set(lineItems.filter(line => line.quantity > 0).map(line => line.skuId)),
  [lineItems]
);
```
`lineItems` is local React state hydrated from `floorWalkLinesQuery.data` but then
updated by every manual qty input. This means any item the operator types a qty for
immediately gets the priority amber highlight — even items they're evaluating, not
confirming.

The priority flag is intended to mark items that were scanned in a previous (or
current session's server-saved) floor walk, to distinguish them from items added to
the order via the catalog. `OrderDetail.tsx` uses the parallel pattern
`importedSkuIds = new Set(floorWalkLinesQuery.data?.filter(...))` which is correct.

**Decision:**
Mirror the `importedSkuIds` pattern from `OrderDetail.tsx`:
```typescript
const prioritySkuIds = useMemo(
  () => new Set(
    floorWalkLinesQuery.data?.filter(l => l.quantity > 0).map(l => l.sku_id) ?? []
  ),
  [floorWalkLinesQuery.data]
);
```

**Consequences:**
- Positive: only server-committed floor walk lines are flagged as priority
- Neutral: items scanned in the current session don't show priority until the session
  is saved (auto-save behaviour on scan is unchanged; this just changes the badge source)
- Acceptable regression risk: low — priority badge is informational, not functional

---

### ADR-3 — Gate tier labels on `kayleeQty !== undefined`

**Context:**
`resolveSkuTier(qty, kayleeQty, isImported)` in `OrderDetail.tsx`:
```typescript
export function resolveSkuTier(qty: number, kayleeQty: number | undefined, isImported: boolean): number {
  if (kayleeQty !== undefined) {
    if (qty > kayleeQty) return 1;        // INCREASED
    if (qty === kayleeQty) return 2;      // KAYLEE
    if (qty > 0) return 3;               // DECREASED
    return 4;                            // GUESS
  }
  if (!isImported && qty > 0) return 1;  // BUG: shows INCREASED for any manual entry
  return getQtyConfidenceTier(qty);      // BUG: shows tier based on qty magnitude
}
```
Without a Kaylee recommendation loaded, operators see INCREASED/DECREASED/KAYLEE
on every row with a non-zero qty, based purely on quantity magnitude. This is
misleading and unrelated to actual Kaylee comparison.

**Decision:**
When `kayleeQty === undefined`, always return tier 4 (GUESS / no badge):
```typescript
export function resolveSkuTier(qty: number, kayleeQty: number | undefined, isImported: boolean): number {
  if (kayleeQty !== undefined) {
    if (qty > kayleeQty) return 1;
    if (qty === kayleeQty) return 2;
    if (qty > 0) return 3;
    return 4;
  }
  return 4;  // No Kaylee recommendation → no tier label
}
```
`getQtyConfidenceTier` fallback is removed from `resolveSkuTier`. It can remain
exported for other uses (if any).

**Consequences:**
- Positive: tier labels are honest; no false signal
- Neutral: operators who haven't loaded Kaylee see no badges at all (this is correct)
- Removed `!isImported && qty > 0 → return 1` — this was never intentional per UX spec

---

### ADR-4 — Qualify recommendations by `riskScore` (not `suggestedQty` alone)

**Context:**
`applyKayleeRecommendations()` in `OrderDetail.tsx`:
```typescript
const sorted = [...sourceSkus]
  .filter((sku) => sku.suggestedQty && sku.suggestedQty > 0)
  ...
```
`suggestedQty` maps from `items.o` which is computed by `cmd/ingest` as
`suggestedQty(velocity, caseSize)`. When ingest runs without sales data (dev env),
velocity = 0 → `items.o = 0` → `suggestedQty = undefined`. After velocitysignalwiring2,
`risk_score` is the canonical recommendation signal; `items.o` is a legacy derived
field. In both dev and post-wiring2 prod, the filter produces an empty list → button
does nothing.

**Decision:**
Change the qualification predicate to OR-combine `suggestedQty` and `riskScore`:
```typescript
const sorted = [...sourceSkus]
  .filter((sku) => (sku.suggestedQty && sku.suggestedQty > 0) ||
                   (sku.riskScore !== undefined && sku.riskScore > 0))
  ...
// Apply qty: use suggestedQty if available, else 1 case (caseSize || 1)
const qty = sku.suggestedQty ?? (sku.caseSize || 1);
```

Verify `riskScore` is mapped in `normalizeItem` (`use_vendor_catalog.ts`):
```typescript
riskScore: typeof row.risk_score === 'number' ? row.risk_score : undefined,
```
This field was added as part of velocitysignalwiring2 normalisation. Confirm it exists
in `ChairSku` type and is passed through.

**Consequences:**
- Positive: button works in dev (risk_score populated by risk-refresh); works in prod
- Positive: intent of velocitysignalwiring2 ("use risk scores for recommendations") is
  now reflected in the UI
- Risk: items with risk_score > 0 but suggestedQty = 0 get `caseSize || 1` applied —
  conservative qty. Operator can adjust.
- Risk: large catalogs may have many items with risk_score > 0. Accept for now; a
  threshold slider can be added in a future iteration.

---

## API Contracts

No API contract changes. `GET /v1/items?vendor_id=X` response shape is unchanged.

`GetItemsByVendorRow` struct already emits:
```json
{
  "i": "SKU-001",
  "o": 4,           // or null/0 when not computed
  "dos_days": 12,   // or null
  "risk_score": 67, // or null
  "vc": "fast",     // velocity class
  ...
}
```

Note: `GetItemsByVendor` SQL does NOT select `i.vc`. This is a pre-existing gap.
**Not in scope for uifix2** — adding `vc` to `GetItemsByVendor` is a separate story.
`use_vendor_catalog.ts` already maps `row.vc` gracefully (`undefined` when absent).

---

## Data Model Changes

None. No migrations. No new DB fields.

---

## Dependencies

| Dependency | Type | Status |
|---|---|---|
| velocitysignalwiring2 | feature | Merged, live in develop |
| `ChairSku.riskScore` type field | type definition | Must confirm exists; add if missing |
| `ChairSku.dosDays` type field | type definition | Must confirm exists; not needed for fix but part of model |

---

## Rollout Strategy

1. All four changes implemented in `fourdogs-central-ui` feature branch
2. PR targets `develop`
3. No backend deployment needed
4. Smoke-test: manually verify all 4 acceptance criteria in dev environment
5. No feature flag required — these are defect fixes, not new features

---

## Testing Strategy

### Unit tests

| File | Test | Assertion |
|---|---|---|
| `catalogTabs.ts` | `buildCatalogTabs` with SE Pet SKUs (no Ruffwear) | Returns no `ruffwear` tab |
| `catalogTabs.ts` | `buildCatalogTabs` with Ruffwear SKUs | Returns `ruffwear` tab |
| `OrderDetail.tsx` | `resolveSkuTier(4, undefined, false)` | Returns 4 (no badge) |
| `OrderDetail.tsx` | `resolveSkuTier(4, 2, false)` | Returns 1 (INCREASED) |
| `OrderDetail.tsx` | `resolveSkuTier(4, 4, false)` | Returns 2 (KAYLEE) |

### Integration / manual

- Open ordering worksheet against SE Pet in dev → verify no Ruffwear/Tractor tabs
- Enter qty on floor walk without prior floor walk data → no priority badge
- Open worksheet before loading Kaylee → no tier labels
- Press "Load Recommendations" with risk_score data → quantities populated

---

## Observability

No new metrics or traces required. These are UI defect fixes.

---

## Open Questions

1. Should `risk_score > 0` threshold for Defect 4 be raised (e.g., `> 20`) to reduce
   the number of items included in a bulk recommendation apply?
2. Should `vc` (velocity class) be added to `GetItemsByVendor` SQL as part of this
   sprint or deferred?
