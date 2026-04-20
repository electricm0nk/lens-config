# Story: transactions-0-01

## Context

**Feature:** transactions
**Sprint:** 0 — Credential Spike
**Priority:** high (gates all implementation)
**Estimate:** S

## User Story

As the operator, I want all eTailPet API open questions documented and the Vault secret provisioned,
so that implementation can begin without auth or endpoint uncertainty.

## Acceptance Criteria

- [ ] eTailPet API auth mechanism documented: type (API key / OAuth / basic auth), header/param name, credential format
- [ ] Trigger endpoint URL and HTTP method confirmed and documented
- [ ] Rate limit or minimum trigger interval documented (or confirmed as absent)
- [ ] Response shape documented: status codes, body fields, job ID or correlation token (if any)
- [ ] API versioning policy noted (if any)
- [ ] Vault secret provisioned at `secret/terminus/fourdogs/etailpet-api` with correct credential field(s)
- [ ] Spike findings committed to `docs/fourdogs/etailpet-api/transactions/etailpet-api-spike.md`
- [ ] All 5 open questions from tech-plan.md answered

## Technical Notes

This is a non-coding spike story. No Go files are produced.

Spike output document (`etailpet-api-spike.md`) must capture:
1. Auth type and credential format
2. Exact trigger endpoint: `{METHOD} {URL}`
3. Request body (if required) — headers, content-type, body schema
4. Expected success response: status code, body
5. Rate limit or quota
6. Job ID / correlation token in response (yes/no; field name if yes)
7. API versioning: is the path versioned? Deprecation notice policy?

If eTailPet API documentation is unavailable, contact eTailPet support directly. Do not begin
Sprint 1 until all acceptance criteria are met.

Vault provisioning command (requires Vault admin access):
```bash
vault kv put secret/terminus/fourdogs/etailpet-api ETAILPET_API_KEY="<key>"
# Adjust field name based on actual auth type discovered
```

## Dependencies

**Blocked by:** none
**Blocks:** transactions-1-01 (client coding requires auth details)

## Definition of Done

- [ ] Code compiles with no errors or warnings (`go build ./...`)
- [ ] `go test ./...` passes
- [ ] `go vet ./...` passes
- [ ] No credentials, tokens, or secrets in code or committed files
- [ ] Structured log events fire correctly for the story's scope
- [ ] Story acceptance criteria verified by author
- [ ] PR reviewed (or self-reviewed for solo work) and merged to `transactions` branch

## Open Questions

None — the purpose of this story is to resolve open questions.
