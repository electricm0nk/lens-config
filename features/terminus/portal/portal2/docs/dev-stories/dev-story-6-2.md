# Story 6.2: MetricsPanel Layout

Status: done

## Story

As a user,
I want the 6 metric cards displayed in a clearly labeled panel,
so that the metrics section is visually distinct from the service cards.

## Acceptance Criteria

1. Exactly 6 `MetricCard` components rendered
2. Panel has visible section heading ("Platform Metrics" or equivalent)
3. Cards arranged in responsive grid (2 columns mobile, up to 3 on wider viewports — CSS grid via inline style token values)
4. Component tests: 6 cards rendered, heading present, each card receives correct metric prop

## Tasks / Subtasks

- [ ] Task 1: Write failing component tests FIRST (TDD) (AC: #4)
  - [ ] `src/__tests__/components/MetricsPanel.test.jsx`:
    - [ ] Exactly 6 `MetricCard` children rendered
    - [ ] Heading element contains "Platform Metrics" (or "PLATFORM METRICS")
    - [ ] Each card receives correct `metric` prop (check by metric id)
    - [ ] No hardcoded hex in panel styles
  - [ ] Run tests — confirm FAIL (red)
- [ ] Task 2: Create `src/components/MetricsPanel.jsx` (AC: #1, #2, #3)
  - [ ] Props: `{ metrics }` (receives `METRICS` array)
  - [ ] Heading: `<h2>PLATFORM METRICS</h2>` or equivalent with `tokens.accent` color
  - [ ] Grid container: `display: 'grid', gridTemplateColumns: 'repeat(auto-fill, minmax(160px, 1fr))', gap: '12px'`
    - Note: `auto-fill` + `minmax` gives responsive behavior (2 col on mobile, 3 on desktop)
  - [ ] Render `METRICS.map(m => <MetricCard key={m.id} metric={m} />)`
  - [ ] Section wrapper: `<section aria-label="Platform Metrics">`
  - [ ] All styles from `useTheme()` tokens
- [ ] Task 3: Add `MetricsPanel` to `App.jsx` below ServiceGrid (AC: #1)
  - [ ] `<MetricsPanel metrics={METRICS} />`
- [ ] Task 4: Run all tests — green (AC: #4)

## Dev Notes

**Target repo:** `TargetProjects/terminus/portal/terminus-portal`
**Depends on:** Story 6.1 — `MetricCard.jsx` and `METRICS` array exist.
**Layout note (AC: #3):** "CSS grid with inline style token values" — this means the grid styles are applied via `style={{ ... }}` using token values (e.g. gap from a token, or just a fixed `12px`). No external CSS files for grid layout.
**Responsive note:** `repeat(auto-fill, minmax(160px, 1fr))` achieves: 2 columns when panel is ~320–480px wide, 3 columns when wider. No media query needed.
**Section heading accessibility:** Use `<section aria-label="Platform Metrics">` with `<h2>` inside. This creates a landmark region for screen readers (tested in Story 7.2).

### Project Structure Notes

New:
```
src/
├── components/
│   └── MetricsPanel.jsx
└── __tests__/components/
    └── MetricsPanel.test.jsx
```
Modified:
- `src/App.jsx` (add `<MetricsPanel metrics={METRICS} />` below ServiceGrid)

### References

- FR4 (6 metric stub cards): `docs/terminus/portal/portal2/epics.md`
- Architecture: `docs/terminus/portal/portal2/architecture.md` — Section "Metrics Panel"
- Story 6.1 (MetricCard prerequisite)
- Story 7.2 (section heading audited for screen reader)

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List
