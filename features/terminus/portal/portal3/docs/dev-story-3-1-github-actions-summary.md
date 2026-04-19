# Story 3.1: Configure monitored GitHub repositories and workflow-run summaries

Status: ready

## Story

As an operator,
I want to see GitHub Actions activity for the most relevant repos,
so that I can get first-pass CI/CD visibility from the portal.

## Acceptance Criteria

1. The monitored repo list is defined in a dedicated config module
2. The first monitored repos include `electricm0nk/terminus-portal`, `electricm0nk/terminus.infra`, and `electricm0nk/lens.config`
3. No GitHub PAT or secret is embedded in browser code
4. The panel summarizes queued, running, failed, and recent successful workflow runs using public API data only
5. Errors are handled without crashing the portal

## Tasks / Subtasks

- [ ] Add a dedicated Actions repo config module
- [ ] Create a panel component that fetches workflow-run data from the public GitHub API
- [ ] Support loading, refresh, empty, and error states
- [ ] Document the limitation around private repos and runner-level detail

## Dev Notes

Primary files:
- `src/config/githubActions.js`
- `src/components/GitHubActionsPanel.jsx`

Constraint:
- This story must not reintroduce the unsafe browser-side PAT approach from the strawman component.
