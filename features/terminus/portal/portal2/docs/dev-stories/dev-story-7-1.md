# Story 7.1: Keyboard Navigation and Focus Management

Status: done

## Story

As a keyboard user,
I want to navigate all interactive elements using Tab and Enter/Space,
so that the portal is fully usable without a mouse.

## Acceptance Criteria

1. Tab from browser address bar cycles through: theme toggle → refresh button → all 8 enabled service cards (in DOM order)
2. Disabled service cards (fourdogs) NOT in tab order (`tabIndex={-1}` or non-interactive element)
3. Pressing Enter or Space on a focused service card navigates to the service URL
4. Focus indicators visible, using `borderActive` token color (not default outline removed)
5. Playwright E2E test verifies full tab cycle reaches all enabled cards

## Tasks / Subtasks

- [ ] Task 1: Install Playwright (AC: #5)
  - [ ] `npm install -D @playwright/test`
  - [ ] `npx playwright install chromium` (headless Chromium for CI)
  - [ ] Create `playwright.config.js` — baseURL `http://localhost:5173`, single chromium project
  - [ ] Add `"test:e2e": "playwright test"` to `package.json`
- [ ] Task 2: Write failing Playwright E2E test FIRST (TDD) (AC: #5)
  - [ ] `e2e/keyboard-nav.spec.js`:
    - [ ] Load portal → Tab from body → lands on theme toggle button
    - [ ] Tab again → lands on refresh button
    - [ ] Tab 8 more times → lands on each enabled service card in sequence
    - [ ] Fourdogs card is NOT focusable (not in tab sequence)
    - [ ] Press Enter on focused ArgoCD card → navigation occurs (URL contains ArgoCD href)
  - [ ] Run `npx playwright test` — confirm FAIL (red)
- [ ] Task 3: Ensure `fourdogs` ServiceCard has `tabIndex={-1}` (AC: #2)
  - [ ] In `ServiceCard.jsx`: when `service.enabled === false`, render `<div tabIndex={-1} ...>` (or `tabIndex` already excluded since it's a `<div>` with no button inside)
  - [ ] Verify no clickable child inside the disabled card is focusable
- [ ] Task 4: Implement focus indicator styles (AC: #4)
  - [ ] In `ServiceCard.jsx` (enabled): add `:focus-visible` outline using `tokens.borderActive`
    - Since inline styles can't use `:focus-visible`, use a `data-focused` state approach OR inject a `<style>` tag at App level with CSS variable reference
    - Recommended: inject a global style block in `App.jsx`:
      ```jsx
      <style>{`
        :focus-visible { outline: 2px solid ${tokens.borderActive} !important; outline-offset: 2px; }
      `}</style>
      ```
  - [ ] Verify focus outline is visible (not `outline: none` anywhere)
- [ ] Task 5: Enable Enter/Space on enabled ServiceCard anchor (AC: #3)
  - [ ] Verify `<a>` elements already handle Enter natively — browsers navigate on Enter for focused `<a>`
  - [ ] No special handler needed for Enter; Space on `<a>` also triggers click in most browsers
- [ ] Task 6: Run Playwright E2E test — green (AC: #5)
  - [ ] Start dev server: `npm run dev &`
  - [ ] Run: `npx playwright test`

## Dev Notes

**Target repo:** `TargetProjects/terminus/portal/terminus-portal`
**Depends on:** Story 5.3 (Header complete with ThemeToggle + RefreshButton) and Story 6.2 (MetricsPanel complete).
**Tab order in DOM:** Ensure Header renders ThemeToggle before RefreshButton (left-to-right). ServiceCard links follow in grid order (left-to-right, top-to-bottom).
**fourdogs disabled card:** It renders as `<div>` (not `<a>`) with no button inside. By default, `<div>` is not focusable. Explicitly set `tabIndex={-1}` as defensive guard (AC: #2).
**Focus visible style injection:** The global `<style>` tag approach using token value is simple and effective for this single-page app. Inject once in `App.jsx` after theme context is available:
```jsx
const { tokens } = useTheme();
// Inject focus style with current theme token
```
**Playwright baseURL:** `http://localhost:5173` (Vite dev server). In CI, start `vite preview` with the built dist.
**8 enabled cards:** All SERVICES except `fourdogs` = 8 cards. Verify tab count in test.

### Project Structure Notes

New:
```
e2e/
└── keyboard-nav.spec.js
playwright.config.js
```
Modified:
- `src/components/ServiceCard.jsx` (tabIndex={-1} on disabled, focus style)
- `src/App.jsx` (global focus style injection)
- `package.json` (add `test:e2e` script)

### References

- NFR5 (accessibility — keyboard navigation): `docs/terminus/portal/portal2/epics.md`
- Architecture: `docs/terminus/portal/portal2/architecture.md` — Section "Accessibility"
- Story 7.2 (screen reader audit follows keyboard audit)

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List
