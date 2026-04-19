# Story 4.2: Rebaseline tests and build verification for portal3

Status: ready

## Story

As a developer,
I want the updated portal behavior verified by the current test and build pipeline,
so that the portal3 changes are safe to merge and release.

## Acceptance Criteria

1. `npm test` passes in the portal repo
2. `npm run build` passes in the portal repo
3. Updated tests cover the new grouping model
4. FinalizePlan artifacts remain aligned with the implementation-ready scope

## Tasks / Subtasks

- [ ] Run `npm test`
- [ ] Run `npm run build`
- [ ] Update any component/integration tests broken by the new grouping and layout model
- [ ] Confirm the portal3 plan artifacts still reflect delivered scope

## Dev Notes

Primary verification commands:
- `npm test`
- `npm run build`

Target repo:
- `TargetProjects/terminus/portal/terminus-portal`
