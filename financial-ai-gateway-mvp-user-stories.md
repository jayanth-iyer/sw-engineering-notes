# Financial AI Governance & Compliance Wrapper — MVP User Stories

Brainstorming draft only. Ready to implement from, not an implementation.

---

## Tech Stack (MVP)

- **Backend:** Python 3.12, FastAPI
- **Packaging:** UV with `pyproject.toml` (lockfile via UV; no parallel `requirements.txt` as source of truth)
- **LLM path:** LiteLLM proxy as the only egress to models
- **API shape:** OpenAI-compatible `POST /v1/chat/completions` (non-streaming in MVP)
- **App identity:** Gateway API keys bound to tenant, team label, model allowlist, vendor allowlist, and GL/cost center
- **Metadata DB:** SQLite for local/MVP config (keys, budgets, entitlements, list versions)
- **Secrets / token map:** In-process map for the request lifetime only; do not treat SQLite as a vault
- **DLP:** Regex / rule lists for financial identifiers (no NER model in MVP)
- **MNPI:** Versioned restricted-list file or table (tickers / entity strings, exact + simple normalize)
- **Audit:** Append-only local log (JSONL) with optional S3-compatible hook later; hashes plus **redacted** payload
- **Ops:** Structured logs + read-only JSON APIs; **no** dashboard in MVP
- **Policy stance:** **Fail closed** — if a control or audit persist fails, do not call LiteLLM

---

## MVP Goal

Ship a **single policy gateway** in front of LiteLLM so internal apps can send chat completions only after:

1. The caller is authenticated and bound to a tenant/route policy
2. The model/vendor is allowed and ZDR headers can be applied
3. Financial identifiers are redacted on the way out
4. Restricted-list (MNPI) hits are blocked
5. Every decision (allow, redact, deny, error) is written to an append-only audit log
6. Usage is attributed to a GL/cost center and **hard-stopped** at the monthly cap

Success is: **no unauthenticated or uncontrolled call reaches a provider**, and a reviewer can reconstruct *what was sent (redacted)*, *what was blocked*, and *why*.

Not in this slice: Chinese walls, reversible vault/DR, WORM exam storage, SSO, or an admin UI.

---

## In scope vs later

| In MVP | Later |
|---|---|
| API keys, tenant + team **label on the key** | SSO, full RBAC, org directory |
| Model/vendor allowlist | Dynamic routing, on-prem vs cloud by content class |
| Inject ZDR / training opt-out headers | Contract SLA scraping, vendor audits |
| Regex DLP inbound; same rules on **response** (block or strip) | NER, reversible vault, key rotation, rehydration product |
| Restricted list block (exact identifiers) | NLP “unannounced earnings”, alias graphs, HITL |
| Append-only audit of all decisions | S3 Object Lock / 17a-4 WORM, legal hold UI |
| GL on key + hard monthly cap | 85/90% ServiceNow, FX, invoice true-up |
| Policy admin via config/API (no UI) | React dashboard |
| Non-streaming chat completions | Streaming, tools, multimodal, agents |

---

## MVP Acceptance Criteria

- All model traffic from apps goes through this gateway; apps do not hold provider keys.
- Unauthenticated, unknown-tenant, or spoofable-tenant-only requests are rejected.
- If DLP, restricted-list load, entitlement check, budget check, or audit write fails → **no LiteLLM call**, stable error code.
- Denied and failed requests are audited, not only successful completions.
- Outbound prompts have financial identifiers redacted; responses are scanned with the same rules.
- Restricted-list matches never reach LiteLLM.
- Allowed vendors get required opt-out/ZDR headers; missing entitlement → block.
- Each key has GL/cost-center; over cap → block.
- Audit record includes: tenant, key/app id, policy version, decision, rule ids, model, sampling params, **redacted** prompt/response or hashes + pointer, latency, timestamps, LiteLLM/provider request id when present.
- Client errors are machine-readable (`POLICY_*`, `AUTH_*`, `BUDGET_*`, `VENDOR_*`, `AUDIT_*`).

---

## Out of Scope for MVP

- Immutable WORM across clouds; exam-grade retention product
- Enterprise SSO / RBAC rollout
- Persistent tokenization vault, rehydration as a stored mapping, key rotation, DR
- Information-barrier enforcement beyond a **static team label on the API key**
- Ops dashboard / BI
- Multi-region, streaming, tool-calling, prompt caching
- Workflow tools (Jira/ServiceNow) for budget alerts
- Claiming SEC 17a-4 compliance (logging is **exam-oriented**, not certified)

---

## Error codes (MVP)

Use these in JSON `error.code` (and log the same):

| Code | Meaning |
|---|---|
| `AUTH_INVALID` | Missing/invalid API key |
| `POLICY_MODEL_DENIED` | Model/provider not on allowlist |
| `POLICY_MNPI_BLOCKED` | Restricted list hit |
| `POLICY_DLP_BLOCKED` | Response (or inbound, if you choose block vs redact) failed DLP |
| `VENDOR_NO_ZDR` | Vendor not entitled or headers cannot be applied |
| `BUDGET_EXCEEDED` | Monthly cap hit |
| `AUDIT_UNAVAILABLE` | Could not persist audit; fail closed |
| `UPSTREAM_UNAVAILABLE` | LiteLLM/provider timeout or 5xx |
| `PAYLOAD_TOO_LARGE` | Over max prompt size |

---

## User Stories

### US-01 — Authenticate internal apps and bind policy

```gherkin
Story: Authenticate callers with a gateway API key
Given an internal application holds a gateway-issued API key
And that key is bound to tenant_id, team_label, allowed_models, allowed_vendors, and gl_code
When the app calls the gateway without a valid key, or with only a tenant header and no key
Then the gateway must reject the request with AUTH_INVALID
And it must not call LiteLLM
And it must write an audit record for the denial
```

**Notes:** Tenant ID is never sufficient alone. Team label is a string on the key (not a live HR feed).

---

### US-02 — OpenAI-compatible non-streaming chat API

```gherkin
Story: Apps send chat completions through a stable API
Given a valid API key
When the app POSTs to /v1/chat/completions with a model and messages (stream=false or omitted)
Then the gateway must apply all policies and, if allowed, forward via LiteLLM
And it must return an OpenAI-shaped completion or a JSON error with a stable code
When stream=true
Then the gateway must reject with a clear unsupported error (MVP)
```

---

### US-03 — Route only to approved models and vendors

```gherkin
Story: Secure routing to approved LLM providers
Given the API key has an allowed model list and allowed vendor list
When a user submits a prompt through the gateway
Then the system must resolve the requested model to a vendor/route
And it must call LiteLLM only if that pair is allowed
And it must reject unapproved models/providers with POLICY_MODEL_DENIED
```

---

### US-04 — Inject and verify ZDR / retention headers

```gherkin
Story: Enforce vendor retention policy on egress
Given a vendor is marked entitled only with ZDR or enterprise retention settings
When the gateway builds the outbound request
Then it must inject the configured opt-out / ZDR headers for that vendor
And it must block with VENDOR_NO_ZDR if the vendor is not entitled or headers cannot be applied
And it must not call LiteLLM in that case
```

---

### US-05 — Redact financial identifiers on inbound prompts

```gherkin
Story: Detect and redact financial identifiers before outbound calls
Given a prompt contains account numbers, SWIFT/BIC, ABA routing, CUSIP/ISIN, or similar configured patterns
When the request is processed
Then the gateway must detect those patterns with domain regex/rules
And it must replace them with opaque placeholders before LiteLLM
And it must keep the original values only in memory for this request (no persistent vault in MVP)
And the audit log must store the redacted prompt, not the raw identifiers
```

**MVP cut:** no durable token vault; placeholders are per-request. Do not log raw PAN-like values.

---

### US-06 — Scan model responses for the same identifiers

```gherkin
Story: Stop sensitive data leaving in the model response
Given a completion returns from LiteLLM
When the gateway inspects the response text
Then it must run the same financial-identifier rules
And if a match is found it must strip or block per config (MVP default: block with POLICY_DLP_BLOCKED)
And it must audit the decision
And it must not return raw matched identifiers to the client if blocked
```

---

### US-07 — Block restricted-list (MNPI) content

```gherkin
Story: Block prompts that hit the active restricted list
Given a versioned restricted list of tickers and entity strings is loaded
When the gateway evaluates the prompt (and, if cheap, the intended model route)
Then it must match with agreed rules (normalize case; exact ticker/token boundaries — not “understands earnings”)
And on a hit it must block before LiteLLM with POLICY_MNPI_BLOCKED
And it must return the code plus a non-leaky message (do not echo the full watch list)
And it must audit list_version and matched rule ids
```

**Out of this story:** NLP for “unannounced earnings,” fuzzy company nicknames, human override.

---

### US-08 — Fail closed when controls cannot run

```gherkin
Story: Unavailable controls must not fail open
Given DLP rules, restricted list, entitlement config, or the audit sink cannot be used
When a request arrives
Then the gateway must reject the request
And it must not call LiteLLM
And it must use AUDIT_UNAVAILABLE or a control-specific code
```

Include: LiteLLM success + audit persist failure → do not return the completion to the client as “success”; define retry/dead-letter in implementation notes (still fail closed to the caller).

---

### US-09 — Append-only audit for every decision

```gherkin
Story: Capture tamper-evident-enough audit records
Given any request is accepted by the HTTP layer
When a policy decision is made (allow, redact-and-allow, deny, error)
Then the system must append a record with tenant, app/key id, policy_version,
  decision, rule_ids, model, hyperparameters, prompt_hash, response_hash,
  redacted prompt/response pointers or bodies, latency, timestamps
And blocked MNPI/DLP/vendor/budget/auth cases must be recorded too
And MVP storage is local append-only JSONL (production hook: S3-compatible, not full WORM)
```

**Honesty bar:** tamper-evident *enough* for an internal MVP (append-only, hashes). Not “17a-4 certified.”

---

### US-10 — Cost center / GL on every call

```gherkin
Story: Attribute spend to a GL code
Given the API key has a gl_code / cost_center
When a request is allowed through
Then usage (tokens from LiteLLM response, or estimated if missing) is recorded against that GL for the calendar month
And requests with no GL on the key are rejected at config/auth time (cannot call)
```

---

### US-11 — Hard monthly budget cap

```gherkin
Story: Stop spend at the configured cap
Given a cost center has a monthly LLM budget
When cumulative attributed cost would exceed the cap
Then the gateway must reject with BUDGET_EXCEEDED before LiteLLM
And it must audit the denial
```

**Deferred:** 85%/90% alerts and ticket workflows. Optional later: log a warning metric at 85% without blocking.

---

### US-12 — Operators publish versioned policy data

```gherkin
Story: Restricted lists, entitlements, and budgets are versioned
Given an operator can update config via file or admin API (authenticated separately from app keys)
When they publish a new restricted list, vendor entitlement, allowlist, or budget
Then the gateway must record a policy_version
And subsequent decisions must stamp that version on the audit log
And a failed load of policy must fail closed (US-08)
```

No React UI in MVP.

---

### US-13 — Upstream failure without double-charge ambiguity

```gherkin
Story: LiteLLM or provider outage is explicit
Given policies passed and the gateway calls LiteLLM
When the upstream times out or returns 5xx
Then the client gets UPSTREAM_UNAVAILABLE
And the audit record shows the attempt and outcome
And MVP does not auto-retry non-idempotent completions
```

---

### US-14 — Payload limits

```gherkin
Story: Bound prompt size
Given a maximum request size is configured
When messages exceed that limit
Then the gateway rejects with PAYLOAD_TOO_LARGE
And it does not call LiteLLM
And it audits the rejection (store hashes/size, not a huge raw body)
```

---

## Explicitly deferred stories (do not build in first slice)

Keep them so the backlog stays honest.

- **Information barriers (private vs public side)** — needs a real attribute source; key `team_label` is not a Chinese wall.
- **Persistent reversible vault + response rehydration** — key management, reconstruct-for-exam vs never-egress.
- **Ops dashboard** for rejects, spend, drill-in — use JSONL + `GET /admin/audit` later if needed.
- **Budget escalation workflows** at 85/90%.
- **Full WORM / multi-cloud immutability.**

---

## Suggested build order (when you start coding)

1. FastAPI + UV + OpenAI-compatible stub + API keys + `AUTH_INVALID`
2. LiteLLM forward for allowlisted models only
3. Fail-closed audit JSONL on every path
4. Regex DLP inbound + response scan
5. Restricted list
6. ZDR headers + vendor entitlement
7. GL + hard cap
8. Admin/config versioning

---

## Open decisions (resolve before coding, not during)

1. **Response DLP default:** block vs strip-and-return.
2. **Audit body:** store full redacted text vs hash + off-log pointer.
3. **Token cost:** LiteLLM usage object only, or your own price table.
4. **Placeholder format:** e.g. `[FIN_ID_1]` so models are less confused.
5. **Who holds LiteLLM/provider keys:** gateway only (recommended).
