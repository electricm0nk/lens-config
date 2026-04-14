---
initiative: fourdogs-prototype-sepordering
track: quickdev
phase: devproposal
status: implementation-ready
created: 2026-03-30
author: CrisWeber
---

# Tech Spec: fourdogs-prototype-sepordering

**Feature:** `sep-ordering` — Southeast Pet ordering app for Betsy  
**Service:** `fourdogs-prototype`  
**Domain:** `fourdogs`  
**Track:** quickdev  
**Target repo:** `electricm0nk/fourdogs` (local: `TargetProjects/fourdogs/prototype/fourdogs`)

---

## What We're Building

A tablet-friendly, offline-tolerant ordering app (`app.html`) hosted on Google Drive, loaded by the existing `fourdogsloader.hintzmann.net` PWA.

Betsy opens her iPad, walks the floor, checks off what she doesn't need, adjusts quantities for what she does, and hits "Compile Order" — a CSV appears that she uploads directly to Southeast Pet. No internet loss during the floor walk ever costs her work. No Claude required in the ordering process.

The entire app is a **single HTML file** (`app.html`) Todd can update by overwriting a file in Google Drive — no deployment pipeline, no CI/CD, no kubernetes changes.

---

## Context

- The loader (`loader.html`) is already deployed at `https://fourdogsloader.hintzmann.net`.
- The loader fetches a named file from a Drive FILE_ID and boots it. Today that file is a placeholder; this feature makes it the ordering app.
- Item data (`sep_items.json`) is a second Drive file Todd maintains by running the vendor data pipeline.
- Everything user-facing lives in `app.html`. Todd owns the Drive files. Betsy owns the iPad.

---

## Architecture Decisions

| Decision | Choice | Rationale |
|---|---|---|
| App distribution | Single `app.html` on Google Drive | Todd can update without touching k8s; existing loader pattern |
| Data source | `sep_items.json` on Google Drive (Drive API) | Same Drive API key already in use; Todd controls refresh |
| State persistence | `localStorage` (`fourdogs_sep_order_<date>`) | Offline-tolerant; no backend needed; survives tab close |
| CSV export | Client-side (Blob + anchor click) | No network during "Compile Order" — Betsy's constraint |
| Styling | Single-file CSS, system fonts, large tap targets | iPad-first; no build step; fast load |
| Data fetching | Drive API v3 `?alt=media` (same pattern as loader) | Reuses existing API key + referrer restriction |

### Data Contract: `sep_items.json`

```json
[
  {
    "sku": "SEP-12345",
    "name": "Fromm Gold Adult",
    "brand": "Fromm",
    "category": "Dry Dog",
    "unit": "30lb bag",
    "suggested_qty": 3,
    "velocity": "high",
    "always_order": false,
    "notes": ""
  }
]
```

Fields:
- `sku` — Southeast Pet item ID (used in CSV)
- `name` — display name
- `brand` — grouping key (items grouped by brand in UI)
- `category` — secondary grouping
- `unit` — order unit (e.g. "30lb bag", "case/12")
- `suggested_qty` — pre-filled quantity (from velocity/history)
- `velocity` — `"high"`, `"medium"`, `"low"` (visual indicator, no blocking logic)
- `always_order` — boolean; items Betsy wants shown even if velocity=0
- `notes` — free text; shown in smaller text below item name

### State Schema: localStorage

```json
{
  "date": "2026-03-30",
  "fetched_at": "2026-03-30T14:00:00Z",
  "items": [
    { "sku": "SEP-12345", "qty": 3, "skip": false }
  ]
}
```

Key: `fourdogs_sep_order_<date>` (date = YYYY-MM-DD of session).  
All writes are immediate on every user interaction — no "save" button.

### CSV Export Format

```
Item Number,Description,Qty
SEP-12345,"Fromm Gold Adult 30lb",3
SEP-67890,"Fromm Gold Puppy 15lb",2
```

Southeast Pet import format. Only rows where `skip = false` and `qty > 0` are included.

---

## User Flow

```
1. Betsy opens fourdogsloader.hintzmann.net on iPad
2. Loader fetches app.html from Drive; app boots
3. App checks localStorage for today's session → restore if present
4. App fetches sep_items.json from Drive (API key reused)
5. Items render: grouped by brand, sorted by priority (always_order first, then velocity desc)
6. Betsy walks floor:
   a. Tap checkbox → mark item as "skip" (skip=true, row visually dimmed)
   b. Tap +/− → adjust quantity (stored immediately to localStorage)
7. At any point: internet failure, tab close, iPad sleep → reopen, state restored
8. Tap "Compile Order":
   a. Generate CSV from non-skipped items with qty > 0
   b. Open as downloadable file (Blob URL)
   c. Betsy copies/shares or downloads → uploads to Southeast Pet portal
9. "New Session" clears state for next ordering cycle
```

---

## Implementation Stories

### Story 1 — Data Loading and Item Rendering

**Goal:** Fetch `sep_items.json` from Drive; render items grouped by brand with velocity badges and suggested quantities pre-filled.

**Scope:**
- Fetch `sep_items.json` using same Drive API pattern as loader (`?alt=media&key=...&t=...`)
- Group items by brand (alphabetical, with always_order items pinned to top within each group)
- Show velocity badge (🔴 low / 🟡 medium / 🟢 high) next to item name
- Render quantity input (default = `suggested_qty`) and skip checkbox per item
- Show loading state while fetching; show error state if Drive fetch fails
- Restore from localStorage if a session exists for today's date

**Acceptance Criteria:**
- Items visible within 3s on a normal connection
- Loading error shows human-readable message (not JS stack trace)
- Session restores correctly on page reload mid-session
- Suggested quantities match the JSON data

---

### Story 2 — Floor Walk Interaction and Persistence

**Goal:** All user interactions persist immediately to localStorage; no data loss on any interruption.

**Scope:**
- Checkbox taps set `skip=true/false` → immediately write to localStorage
- `+` / `−` buttons update qty (min 0, max 99) → immediately write to localStorage
- Numeric input on qty field (direct entry for power users)
- Visual treatment for skipped items (dimmed, struck-through name)
- Running count: "X items · Y units selected" updates in real-time
- "New Session" button clears today's localStorage key after confirmation prompt

**Acceptance Criteria:**
- Close and reopen tab mid-floor-walk → all state exactly preserved
- Qty + / − buttons each work with a single tap (no double-tap required)
- Skipped items can be unskipped (tap checkbox again)
- New session confirmation: "Start fresh? Your current order will be cleared." + Cancel/Confirm

---

### Story 3 — CSV Export

**Goal:** "Compile Order" generates a correctly formatted CSV that Betsy can upload to Southeast Pet.

**Scope:**
- Only include items where `skip=false` AND `qty > 0`
- CSV headers: `Item Number,Description,Qty`
- Each row: sku, quoted name, qty
- Generate as Blob URL → trigger browser download (or share sheet on iOS)
- Filename: `fourdogs-sep-order-<YYYY-MM-DD>.csv`
- "Compile Order" button visible at top (sticky) once items are loaded
- Button is disabled until at least one item has qty > 0

**Acceptance Criteria:**
- CSV opens in Numbers/Excel without import wizard (correct format)
- Skipped items and zero-qty items excluded from CSV
- Filename includes today's date
- Button disabled when no items selected; re-enables when qty > 0 on any item
- iOS share sheet appears (allows AirDrop to laptop, or "Open in Files")

---

## Drive File Setup

Two files live in a shared Google Drive folder Todd manages:

| File | Drive Role | Update frequency |
|---|---|---|
| `app.html` | The ordering app | When feature ships; ongoing updates |
| `sep_items.json` | Item list + velocity data | Before each ordering cycle (Todd runs pipeline) |

`app.html` FILE_ID is what `loader.html` already has hardcoded. Today it loads a placeholder — after this feature ships, it loads the ordering app.

`sep_items.json` FILE_ID is a new Drive file ID; it will be hardcoded in `app.html`.

Both files are shared "anyone with link can view". The same API key (restricted to `fourdogsloader.hintzmann.net/*`) is reused.

---

## Out of Scope (This Feature)

- EtailPet API / automated data pull (belongs to `data-refresh` feature)
- Multi-device sync or backend persistence
- Orders for other distributors (OC Raw, Fromm direct) — future features on `fourdogs-ordering` service
- Lauren / Julia rollout
- EtailPet writeback
- QuickBooks integration

---

## Implementation Notes

- `app.html` is a single file (HTML + inline CSS + inline JS). No build tools. No dependencies. Opens and works in any modern browser.
- Minimum tap target: 44×44px (Apple HIG for iPad)
- Brand headers are sticky within their section so Betsy can always see which brand she's in
- Test with actual `sep_items.json` data before handing to Betsy — velocity suggestions must feel right
- The FILE_ID for `sep_items.json` will be provided by Todd before implementation of Story 1

---

## Definition of Done

- [ ] `app.html` uploaded to Google Drive and loader FILE_ID updated to point to it
- [ ] `sep_items.json` uploaded to Google Drive with current Southeast Pet item list
- [ ] Betsy can complete a full ordering session end-to-end on iPad
- [ ] CSV export verified against Southeast Pet import format
- [ ] localStorage persistence verified (close tab mid-session, reopen → state intact)
- [ ] Error state verified (toggle airplane mode, reload → graceful error)
