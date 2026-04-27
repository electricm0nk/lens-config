# Story: sales-0-01

## Context

**Feature:** fourdogs-etailpet-api-sales
**Sprint:** 0 — www Validation Spike
**Priority:** high (gates all implementation)
**Estimate:** S

## User Story

As the operator, I want the `www.etailpet.com` legacy report trigger validated with existing
Vault credentials and the sales-line email attachment schema documented, so that implementation
can begin with full confidence in the auth flow and response contract.

## Acceptance Criteria

- [ ] `www` legacy token endpoint accepts existing Vault CLIENT_ID/CLIENT_SECRET; access_token returned and documented
- [ ] `GET /fourdogspetsupplies/api/v1/transaction-report/?report_type=sales-line` returns `200 OK`
- [ ] Sales-line report email received at Four Dogs retailer inbox; subject line documented
- [ ] xlsx attachment opened; column headers (first row) documented verbatim
- [ ] `TO_EMAIL` query param behavior documented: default (retailer address) vs. explicit override
- [ ] Spike findings committed to `docs/fourdogs/etailpet-api/fourdogs-etailpet-api-sales/sales-api-spike.md`
- [ ] emailfetcher routing impact assessed: can existing rules distinguish this email from transaction-report emails?

## Technical Notes

This is a non-coding spike story. No Go files are produced.

**www legacy token request (query-param OAuth2):**
```
POST https://www.etailpet.com/fourdogspetsupplies/oauth2/token/
  ?grant_type=client_credentials
  &client_secret={ETAILPET_CLIENT_SECRET}
  &client_id={ETAILPET_CLIENT_ID}

Expected response: {"access_token": "...", "token_type": "Bearer", "expires_in": 36000}
```

**Sales-line trigger request:**
```
GET https://www.etailpet.com/fourdogspetsupplies/api/v1/transaction-report/
  ?report_type=sales-line
  &file_format=xlsx
  &start_date=YYYY-MM-DD    ← note www uses YYYY-MM-DD; pos uses MM/DD/YYYY
  &end_date=YYYY-MM-DD

Headers:
  Authorization: Bearer {access_token}
  Content-Type: application/json
```

**Spike output document** (`sales-api-spike.md`) must capture:
1. Token endpoint response shape (confirm `access_token`, `token_type`, `expires_in`)
2. Trigger response: status code, response body (if any)
3. Email subject line (exact)
4. xlsx attachment: column headers verbatim from first row
5. `TO_EMAIL` behavior
6. emailfetcher routing notes

**Testing discipline (inherited):** Make at most ONE live trigger call per day against the production
API. Confirm token validity first with a no-op test call if possible before triggering the export.
There is no sandbox environment.

**Note:** Do NOT test sibling `report_type` values (e.g. `basic`, `orders`) in Sprint 0. Only
confirm `sales-line`. Additional report types are out of scope for this feature.

## Dependencies

**Blocked by:** none (Vault credentials already provisioned at `secret/terminus/fourdogs/etailpet-api`)
**Blocks:** sales-1-01 (LegacyClient requires confirmed token form and trigger behavior)

## Definition of Done

- [ ] All acceptance criteria verified
- [ ] Spike doc committed to `docs/fourdogs/etailpet-api/fourdogs-etailpet-api-sales/sales-api-spike.md`
- [ ] emailfetcher routing impact noted in spike doc
- [ ] No credentials, tokens, or secrets in committed files
- [ ] PR reviewed and merged to `fourdogs-etailpet-api-sales` branch

## Spike Fail Path

If any confirmation item cannot be obtained within the **3-day timebox**:

1. **Do not start Sprint 1.** Sprint 1 is hard-blocked on this story's acceptance.
2. Document the failure point in `sales-api-spike.md` with exact error/evidence.
3. Open an eTailPet support ticket (contact: Todd Hintzmann) describing the specific auth or routing failure.
4. Set story status to `blocked`; update `sprint-plan.md` Sprint 0 with the blocking reason.
5. Re-evaluate Sprint 1 start date only after eTailPet support responds.

**Partial confirmation is not acceptable.** All six acceptance criteria must be verified.
