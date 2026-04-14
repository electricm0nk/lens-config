# Story 3.2: ServiceGrid with Category Grouping

Status: done

## Story

As a user,
I want services grouped by category with labeled sections,
so that I can quickly find a service by its operational domain.

## Acceptance Criteria

1. Services grouped by category with visible heading above each group
2. Services within a group rendered in the order they appear in `SERVICES`
3. A category section renders ONLY if at least one service belongs to it
4. Component tests verify: correct grouping, correct heading per category, empty category not rendered

## Tasks / Subtasks

- [ ] Task 1: Write failing component tests FIRST (TDD) (AC: #4)
  - [ ] `src/__tests__/components/ServiceGrid.test.jsx`:
    - [ ] Services grouped by category — `infra` group contains ArgoCD, Vault, k3s-Dashboard, Proxmox, Consul
    - [ ] Heading text matches category (e.g. "INFRA", "DATA", "AI", "APP")
    - [ ] Pass a SERVICES subset with empty `app` category → no "APP" heading rendered
    - [ ] Order within group matches SERVICES array order
  - [ ] Run tests — confirm FAIL (red)
- [ ] Task 2: Create `src/components/ServiceGrid.jsx` (AC: #1, #2, #3)
  - [ ] Accept `{ services, statusMap }` props (statusMap defaults to `{}` for now)
  - [ ] Group services: `services.reduce((acc, s) => { acc[s.category] = [...(acc[s.category] || []), s]; return acc; }, {})`
  - [ ] Preserve category order (use CATEGORY_ORDER constant: `['infra', 'platform', 'data', 'ai', 'app']`)
  - [ ] For each category with ≥1 service: render `<section>`, category `<h2>`, `ServiceCard` list
  - [ ] Skip category if no services belong to it (AC: #3)
  - [ ] All styles from `useTheme()` tokens
- [ ] Task 3: Replace inline ServiceCard loop in `App.jsx` with `<ServiceGrid>` (AC: #1)
  - [ ] Pass `services={SERVICES}` to ServiceGrid
- [ ] Task 4: Run all tests — green (AC: #4)

## Dev Notes

**Target repo:** `TargetProjects/terminus/portal/terminus-portal`
**Depends on:** Story 3.1 — `ServiceCard` and populated `SERVICES` must exist.
**Categories in use** (from SERVICES): `infra`, `data`, `ai`, `app`. `platform` currently unused — no section rendered.
**Category headings:** Suggest uppercased display — `infra` → "INFRASTRUCTURE" or "INFRA". Keep it short.
**statusMap:** `{ [serviceId]: STATUS }` — passed from App-level polling state (Story 4.3). For now defaults to `{}`, which means each ServiceCard receives `status={undefined}` (handled in Story 4.3).

### Project Structure Notes

New:
```
src/
├── components/
│   └── ServiceGrid.jsx
└── __tests__/components/
    └── ServiceGrid.test.jsx
```
Modified:
- `src/App.jsx` (replace ServiceCard loop with ServiceGrid)

### References

- FR1 (service registry with categories): `docs/terminus/portal/portal2/epics.md`
- Architecture: `docs/terminus/portal/portal2/architecture.md` — Section "Service Registry"
- Story 3.1 (ServiceCard prerequisite)
- Story 4.3 (statusMap wired from polling loop)

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List
