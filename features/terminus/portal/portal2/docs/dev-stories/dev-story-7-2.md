# Story 7.2: Screen Reader Labels and Semantic HTML Audit

Status: done

## Story

As a screen reader user,
I want all status indicators, buttons, and icons to have meaningful labels,
so that the portal content is fully audible without visual context.

## Acceptance Criteria

1. Every `StatusIndicator` has `aria-label` matching "Status: Online/Unreachable/Checking/No health check"
2. Every service card icon `<img>` has non-empty `alt` attribute matching the service name
3. Theme toggle `aria-label` describes the action ("Switch to Terminal theme" / "Switch to Space Ops theme")
4. Refresh button `aria-label` is "Refresh health status"
5. Metrics section heading and each metric card label are screen-reader readable
6. No `<div>` or `<span>` used for interactive elements that should be `<button>` or `<a>`
7. Playwright E2E: checks `aria-label` on StatusIndicators for at least 3 services
8. axe-core scan returns zero violations at WCAG 2.1 AA level

## Tasks / Subtasks

- [ ] Task 1: Install axe-core (AC: #8)
  - [ ] `npm install -D @axe-core/playwright`
  - [ ] (Or use `axe-playwright` / `playwright-axe` — choose one)
- [ ] Task 2: Write failing Playwright E2E test FIRST (TDD) (AC: #7, #8)
  - [ ] `e2e/accessibility.spec.js`:
    - [ ] Load portal → run axe-core → assert zero violations (WCAG21AA)
    - [ ] Find `[role="status"]` for ArgoCD → assert `aria-label === "Status: Online"` (or CHECKING/UNREACHABLE)
    - [ ] Find `[role="status"]` for Vault → assert aria-label is one of the valid values
    - [ ] Find `[role="status"]` for Semaphore → same check (3 services verified)
    - [ ] Theme toggle button → `aria-label` starts with "Switch to"
    - [ ] Refresh button → `aria-label === "Refresh health status"`
  - [ ] Run tests — confirm FAIL (red) [axe may find violations initially]
- [ ] Task 3: Semantic HTML audit — fix any `<div>`/`<span>` used as interactive (AC: #6)
  - [ ] Search: `grep -rn 'onClick' src/components/ | grep -v 'button\|a '` — identify violations
  - [ ] Fix any `<div onClick>` → replace with `<button>` (styled to look like the original)
  - [ ] Exception: disabled fourdogs card is `<div>` but NOT interactive — OK
- [ ] Task 4: Verify all aria-labels (AC: #1–#4)
  - [ ] `StatusIndicator`: aria-labels already set in Story 4.1 — verify exact strings
  - [ ] Icon `<img>`: alt already set in Story 3.1 — verify non-empty, matches service name
  - [ ] ThemeToggle: aria-label already set in Story 2.3 — verify exact strings
  - [ ] RefreshButton: aria-label already set in Story 4.4 — verify exact string
- [ ] Task 5: Verify metrics section accessibility (AC: #5)
  - [ ] `MetricsPanel` has `<section aria-label="Platform Metrics">` — set in Story 6.2
  - [ ] Each `MetricCard` label is in a `<p>` or `<span>` readable by screen reader (not hidden)
  - [ ] "WIRE TO [SOURCE] TO ENABLE" text is visible text (not aria-hidden)
- [ ] Task 6: Fix all axe-core violations (AC: #8)
  - [ ] Run axe report — address each violation category
  - [ ] Common fixes: color contrast (ensure token colors meet 4.5:1 ratio), missing lang attribute on `<html>`, landmark roles
  - [ ] Add `lang="en"` to `<html>` in `index.html` if missing
- [ ] Task 7: Run Playwright E2E test — green (AC: #7, #8)
  - [ ] `npx playwright test e2e/accessibility.spec.js`
  - [ ] Zero axe violations
  - [ ] aria-label assertions pass for 3+ status indicators

## Dev Notes

**Target repo:** `TargetProjects/terminus/portal/terminus-portal`
**Depends on:** Story 7.1 (Playwright configured, all components exist).
**axe-core integration:**
```js
import AxeBuilder from '@axe-core/playwright';

test('no WCAG 2.1 AA violations', async ({ page }) => {
  await page.goto('/');
  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
    .analyze();
  expect(results.violations).toEqual([]);
});
```
**Color contrast:** The Terminal theme uses `#00ff41` text on `#0a0f0a` background — contrast ratio ~10:1 ✅. Space Ops `#e0f0ff` on `#0d1b2a` — ~8:1 ✅. Status tokens must also meet 4.5:1 on card backgrounds — verify during fix step.
**`lang` attribute:** Add `lang="en"` to `<html>` in `index.html` — common axe violation if missing.
**Note on StatusIndicator aria-labels:** These were already implemented in Story 4.1. This story verifies them via E2E rather than unit test — cross-checking the implementation against running browser behavior.

### Project Structure Notes

New:
```
e2e/
└── accessibility.spec.js
```
Modified (fixes only — should be minimal if earlier stories followed specs):
- `public/index.html` or `index.html` (add `lang="en"` to `<html>`)
- Any component with `<div onClick>` → `<button>` (if violations found)

### References

- NFR5 (accessibility — WCAG 2.1 AA): `docs/terminus/portal/portal2/epics.md`
- Architecture: `docs/terminus/portal/portal2/architecture.md` — Section "Accessibility"
- Story 4.1 (StatusIndicator aria-labels)
- Story 2.3 (ThemeToggle aria-label)
- Story 4.4 (RefreshButton aria-label)
- Story 6.2 (MetricsPanel section landmark)
- Story 7.1 (Playwright setup prerequisite)

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List
