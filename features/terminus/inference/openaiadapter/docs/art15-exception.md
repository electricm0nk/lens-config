---
initiative: terminus-inference-openaiadapter
article: org-art-15
status: approved
approver: ToddHintzmann
approved_date: '2026-04-07'
expiry_date: '2027-04-07'
---

# Art. 15 AI Safety Exception — terminus-inference-openaiadapter

**Org Constitution Article 15: AI Safety Primacy**
**Exception filed:** 2026-04-07
**Expiry:** 2027-04-07 (annual review required)
**Approver:** ToddHintzmann

---

## 1. Decision Statement

The `terminus-inference-openaiadapter` initiative registers OpenAI as a second provider in the terminus inference gateway. This requires forwarding user inference request payloads (prompt messages) to an external third-party AI provider (OpenAI, operated by OpenAI, L.L.C.) over the public internet.

**Risk being accepted:** User prompt data leaves the terminus local infrastructure boundary and is processed by an external AI system not under operator control.

**Reason the work must proceed despite the risk:** The terminus inference platform currently has a single provider (Ollama, local). Any Ollama outage represents a total loss of inference capability for all downstream platform consumers, including fourdogs central. The OpenAI adapter introduces a cloud fallback path that restores availability during local outages. The availability risk of Ollama-only operation — no fallback, no resilience — is judged to outweigh the data exposure risk of a gated, routing-policy-controlled OpenAI path. The risk is bounded (interactive workloads only, per routing policy), known in advance, and mitigable.

---

## 2. Threat and Impact Analysis

| Threat | Failure Mode | Impact Scope |
|--------|-------------|--------------|
| **Data exposure** | Prompt content is sent to OpenAI and processed under OpenAI's data handling policies. Sensitive or proprietary content in prompts could be retained, used for model training (subject to API terms), or exposed in a breach of OpenAI's systems. | Users of the platform whose data appears in interactive inference requests. |
| **Supplier lock-in** | Growing reliance on OpenAI as a fallback may create implicit dependency. Future OpenAI pricing changes, API deprecations, or terms changes could force reactive re-architecture. | Platform maintainability; potential cost or availability impact if OpenAI behavior changes. |
| **Availability dependency** | The fallback path inherits OpenAI's availability profile. If OpenAI is down during an Ollama outage, the platform has no inference path. | Total inference unavailability during overlapping outages. |
| **API key compromise** | If `GATEWAY_OPENAI_API_KEY` is leaked (logs, misconfigured pod, git commit), an attacker can make requests against the key, incur cost, and potentially access data sent through the key. | Financial impact; potential data exposure to third parties. |
| **Prompt injection** | An adversarial user crafting prompts to manipulate OpenAI model behavior or extract system context. | Platform integrity; potential leakage of system prompt structure. |

---

## 3. Mitigations

| Mitigation | Implementation | Status |
|------------|---------------|--------|
| **Data minimisation** | Only `workloadClass: interactive` requests are eligible for OpenAI routing — batch workload data never leaves local infrastructure. Enforced by routing policy (`allowedClasses: [interactive]`). | Binding — defined in PRD FR23, routing config story in devproposal. |
| **Fallback-first routing** | OpenAI is a fallback, not a primary. Routing engine prefers Ollama. OpenAI is only selected when Ollama is unavailable or exhausted. Limits routine exposure. | Binding — defined in routing config priority field. |
| **API key via Vault/ESO** | `GATEWAY_OPENAI_API_KEY` is delivered exclusively via External Secrets Operator from Vault. Never in source code, git history, Helm values files, or logs. | Binding — defined in FR12–FR14, tech-decisions.md §Key Delivery. |
| **Key rotation procedure** | API key is rotatable in Vault without code changes. Pod restarts pick up the new key via ESO resync. Rotation should be performed at any suspected compromise and on a scheduled basis (at minimum annually). | Procedure — operator responsibility. |
| **Telemetry metadata only** | Adapter telemetry logs capture token counts, duration, and outcome. Prompt content is never logged (NFR-D2). Limits secondary exposure surface. | Binding — defined in NFR-D2. |
| **Structured error mapping** | OpenAI error responses are mapped to structured gateway envelopes. Raw OpenAI error bodies (which may echo request content) are never forwarded to callers (NFR-S3 analogue). | Binding — defined in error mapping spec, tech-decisions.md §Error Handling. |
| **TLS-only communication** | All communication with the OpenAI API uses TLS. No plaintext HTTP permitted (NFR-S3). | Binding — enforced by Go `net/http` with `https://` endpoint. |

---

## 4. Expiry Date

**This exception expires on: 2027-04-07.**

Re-evaluation is required before this date. At re-evaluation, the approver must assess:
- Whether the threat model has changed (new OpenAI data handling policies, new prompt injection vectors, etc.)
- Whether the mitigation set remains sufficient
- Whether the usage scope (interactive-only) has been violated or expanded
- Whether an internal alternative to OpenAI has become available that would eliminate the external dependency

If re-evaluation is not completed before the expiry date, the exception is automatically void and all promotion gates are re-blocked until a renewed exception is filed.

---

## 5. Named Approver

**ToddHintzmann** — platform operator and initiative owner.

Approval of this exception confirms:
- The threat and impact analysis is understood
- The mitigations are judged sufficient for the defined scope
- The expiry date and re-evaluation obligation are accepted
- The decision to proceed is deliberate and not made under time pressure alone

---

## 6. Rollback Plan

If the accepted risk materializes (e.g., data breach at OpenAI, key compromise, regulatory requirement change, or policy violation discovered):

1. **Immediate:** Set `providers.openai.enabled: false` in the gateway Helm values and redeploy. The OpenAI adapter will not be registered at startup. Routing engine falls back to Ollama-only. No code changes required.
2. **Key rotation:** Rotate `GATEWAY_OPENAI_API_KEY` in Vault immediately. ESO will resync within its configured interval; a pod restart accelerates pickup.
3. **Audit:** Review gateway telemetry logs for the affected window to understand the scope of requests sent to OpenAI.
4. **Notification:** If sensitive data was in-scope, notify affected parties per applicable data handling obligations.
5. **Re-evaluation:** Do not re-enable the OpenAI adapter until a new exception is filed and the trigger event is fully understood.

Zero downtime impact: Ollama-only operation is the baseline state. Disabling OpenAI reverts to that baseline.

---

## 7. Verification Plan

| Verification | Method | Timing |
|-------------|--------|--------|
| **Key never in logs** | Run gateway log grep for `GATEWAY_OPENAI_API_KEY` value pattern across all log levels in staging before promotion to large. | Pre-large-promotion smoke test. |
| **Batch workloads do not route to OpenAI** | Integration test: send a request with `workloadClass: batch` — verify it does NOT reach the OpenAI adapter (check provider-routing service logs, not just response). | Part of devproposal acceptance test story (FR23). |
| **Telemetry contains no prompt content** | Review 10 sample telemetry log entries in staging — confirm fields are metadata-only (tokens, duration, outcome). | Operator checklist item in smoke test story (FR20). |
| **ESO key delivery confirmed** | In deployed k3s environment, confirm pod env contains `GATEWAY_OPENAI_API_KEY` populated from Vault (not from Helm values). | Part of smoke test story (FR20). |
| **Rollback tested** | In staging, set `providers.openai.enabled: false`, redeploy, confirm gateway starts Ollama-only and logs no OpenAI registration. | Resilience test story (FR21). |
| **Annual re-evaluation** | Calendar entry for 2027-04-07 to re-evaluate this exception. | Approver (ToddHintzmann) responsibility. |
