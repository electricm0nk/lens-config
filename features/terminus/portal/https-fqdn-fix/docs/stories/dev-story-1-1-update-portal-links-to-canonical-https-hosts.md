# Story 1.1: Update Portal Links to Canonical HTTPS Hosts

Status: done

## Story

As an operator,
I want the portal dashboard to use canonical HTTPS hostnames instead of legacy IP-based and deprecated local links,
so that operational navigation stays aligned with current DNS, avoids mixed protocol drift, and removes stale bookmarks from the UI.

## Acceptance Criteria

1. The Proxmox dashboard card in `TargetProjects/terminus/portal/terminus-portal/index.html` points to `https://proxmox.trantor.internal:8006/` in both its `href` and visible URL text.
2. The legacy Vault direct-IP dashboard card using `http://10.0.0.75:8200` is removed from `TargetProjects/terminus/portal/terminus-portal/index.html`.
3. The legacy Fourdogs local dashboard card using `https://fourdogs.terminus.local` is removed from `TargetProjects/terminus/portal/terminus-portal/index.html`.
4. No remaining references to `10.0.0.48:8006`, `10.0.0.75:8200`, or `fourdogs.terminus.local` remain in the rendered dashboard links.
5. The portal change is committed on a dedicated feature branch and pushed to the target repository for review.

## Tasks / Subtasks

- [x] Update the Proxmox card link target and visible URL to the canonical FQDN over HTTPS. (AC: 1)
- [x] Remove the legacy Vault direct-IP card from the infrastructure section. (AC: 2)
- [x] Preserve and include the existing removal of the legacy Fourdogs local card in the final change set. (AC: 3)
- [x] Verify the dashboard file no longer contains the deprecated IP or legacy local host references. (AC: 4)
- [x] Commit the dashboard change on `feature/https-fqdn-fix` and push it to `origin`. (AC: 5)

## Dev Notes

- Initiative track: `hotfix`
- Source plan: `docs/terminus/portal/https-fqdn-fix/architecture.md`
- Target repo: `TargetProjects/terminus/portal/terminus-portal`
- Target branch: `feature/https-fqdn-fix`
- Implementation commit: `d1110c5`

### Validation

- Verified the final diff in `index.html` removes the deprecated Vault and Fourdogs cards.
- Verified the Proxmox link now uses `https://proxmox.trantor.internal:8006/`.
- Verified the HTML file reports no editor errors after the change.

### References

- [Architecture Plan](docs/terminus/portal/https-fqdn-fix/architecture.md)

## Dev Agent Record

### Agent Model Used

GPT-5.4

### Debug Log References

- Portal target repo branch: `feature/https-fqdn-fix`

### Completion Notes List

- Implemented the planned Proxmox FQDN update.
- Removed the legacy Vault direct-IP tile.
- Preserved the in-progress removal of the legacy Fourdogs local tile.
- Pushed the implementation branch for PR creation.
- Opened target-repo PR: `https://github.com/electricm0nk/terminus-portal/pull/1`

### File List

- `TargetProjects/terminus/portal/terminus-portal/index.html`