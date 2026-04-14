# Story 6.1: Metrics Config and MetricCard Component

Status: done

## Story

As a developer,
I want all 6 metrics defined in `src/config/metrics.js` and a `MetricCard` component,
so that the metrics panel renders correctly and the wiring interface is locked in for future data feeds.

## Acceptance Criteria

1. `METRICS` in `src/config/metrics.js` contains exactly 6 entries
2. `MetricCard` with `wired: false` displays: label, `—` value, unit, source badge, "WIRE TO [SOURCE] TO ENABLE"
3. `MetricCard` accepts `metric` prop of shape `{ id, label, unit, source, value, wired }`
4. When `wired: true` and `value` is a number, renders value + unit without "WIRE TO" label
5. All colors and fonts use theme tokens
6. Unit tests: unwired renders `—` + label, wired with value renders value, source badge shows correct source

## Tasks / Subtasks

- [ ] Task 1: Write failing unit tests FIRST (TDD) (AC: #6)
  - [ ] `src/__tests__/components/MetricCard.test.jsx`:
    - [ ] Unwired metric (`wired: false`): displays `—` as value, shows "WIRE TO PROMETHEUS TO ENABLE" (or source name)
    - [ ] Wired metric (`wired: true`, `value: 42`): displays "42" + unit, no "WIRE TO" label
    - [ ] Source badge: `source: 'prometheus'` → "PROMETHEUS"; `source: 'influxdb'` → "INFLUXDB"
    - [ ] No hardcoded hex in rendered styles
  - [ ] Run tests — confirm FAIL (red)
- [ ] Task 2: Populate `src/config/metrics.js` with 6 entries (AC: #1)
  - [ ] Shape: `{ id, label, unit, source, value: null, wired: false }`
  - [ ] Entries:
    - `{ id: 'cpu-load', label: 'CPU Load', unit: '%', source: 'prometheus', value: null, wired: false }`
    - `{ id: 'mem-used', label: 'Memory Used', unit: 'GB', source: 'prometheus', value: null, wired: false }`
    - `{ id: 'net-in', label: 'Net In', unit: 'Mbps', source: 'prometheus', value: null, wired: false }`
    - `{ id: 'net-out', label: 'Net Out', unit: 'Mbps', source: 'prometheus', value: null, wired: false }`
    - `{ id: 'pods-running', label: 'Pods Running', unit: '', source: 'influxdb', value: null, wired: false }`
    - `{ id: 'disk-used', label: 'Disk Used', unit: 'GB', source: 'influxdb', value: null, wired: false }`
- [ ] Task 3: Create `src/components/MetricCard.jsx` (AC: #2, #3, #4, #5)
  - [ ] Props: `{ metric }` of shape `{ id, label, unit, source, value, wired }`
  - [ ] If `wired === false` OR `value === null`: display `—`, show "WIRE TO {SOURCE.toUpperCase()} TO ENABLE"
  - [ ] If `wired === true` AND `value !== null`: display `{value}{unit}`, no "WIRE TO" label
  - [ ] Source badge: `<span>{metric.source.toUpperCase()}</span>` styled with `tokens.textMuted`
  - [ ] All styles from `useTheme()` tokens
- [ ] Task 4: Run all tests — green (AC: #6)

## Dev Notes

**Target repo:** `TargetProjects/terminus/portal/terminus-portal`
**Parallel with E2–E5:** This story only requires Story 1.1 (Vite scaffold + config files). It is independent of the theme and health polling stories.
**TR7:** Metric prop shape is locked: `{ id, label, unit, source, value, wired }`. This is the interface for future data feed wiring — do NOT deviate.
**VAULT_STATUS_DETAIL:** Not a metric card — that's a service health concern documented separately.
**Upgrade path:** When a data source is available, update `wired: true` and `value: <number>`. The MetricCard renders correctly in both states.

### Project Structure Notes

Modified:
- `src/config/metrics.js` (fully populated)

New:
```
src/
├── components/
│   └── MetricCard.jsx
└── __tests__/components/
    └── MetricCard.test.jsx
```

### References

- FR4 (metric stub panel, 6 cards): `docs/terminus/portal/portal2/epics.md`
- TR7 (metric prop shape): `docs/terminus/portal/portal2/epics.md`
- Architecture: `docs/terminus/portal/portal2/architecture.md` — Section "Metrics Panel"
- Story 6.2 (MetricsPanel uses MetricCard)

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List
