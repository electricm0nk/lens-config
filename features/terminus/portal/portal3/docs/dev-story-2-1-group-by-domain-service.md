# Story 2.1: Group services by domain and service

Status: ready

## Story

As an operator,
I want services grouped by Domain and Service,
so that the portal matches how I think about the environment rather than forcing category scanning.

## Acceptance Criteria

1. Services render in grouped sections based on `domain` and `service`
2. Terminus Platform and Terminus AI appear as distinct sections when both have services
3. Fourdogs Central renders as its own section when present
4. Services retain stable ordering within each section
5. Grouping tests assert the new domain/service structure

## Tasks / Subtasks

- [ ] Expand service metadata to include `domain` and `service`
- [ ] Refactor `ServiceGrid` to group by domain/service key
- [ ] Update tests to assert section identity via domain/service grouping
- [ ] Verify Kaylee Agent appears in the AI grouping

## Dev Notes

Primary files:
- `src/config/services.js`
- `src/components/ServiceGrid.jsx`
- `src/__tests__/components/ServiceGrid.test.jsx`
