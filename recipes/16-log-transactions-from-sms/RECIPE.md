# Log transactions from SMS

## A. Identity and ownership

- **Recipe ID:** `nova.finance.log-transactions-from-sms` (proposed until assigned by Skill Port)
- **Name:** Log transactions from SMS
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** finance
- **Tags:** transactions, sms, webhook, bank-alerts, idempotency, finance-hub, review-queue · Vedaxi task type: none (no supported task type covers this today)
- **Summary:** An SMS-forwarder app on the user's phone posts bank and card text messages to an authenticated webhook. The agent normalises the payload, drops anything that is not a money message (one-time codes are never sent to an LLM), and computes a hashed idempotency key. It then extracts the transaction with an LLM and routes it exactly like the email logger: high-confidence matches inside the user's standing approval are logged, everything else goes to the review queue, and duplicates are reconciled.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** A user-installed SMS-forwarder app that can POST JSON (or form data) with a custom header; an HTTPS webhook on the automation hub; a structured-output LLM; the finance hub router, review queue and external-app sync declared in `nova.finance.log-transactions-from-email`. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — the core flow (webhook, normalisation, LLM extraction and router) has run in Nova's reference implementation with test messages (catalogue status: Built). Not every control described here has evidence yet (Section P lists the tests still needed). Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.sms.ingest` | Capability | Receive forwarded SMS over an authenticated webhook | high | UNVERIFIED |
| `nova.sms.ingest.receive` | Operation (inbound endpoint) | Authenticate, size-check and normalise one forwarded message | high | UNVERIFIED |
| `nova.sms.ingest.register` | Operation (create) | Insert or look up the idempotency key in the ingest ledger | low | UNVERIFIED |
| `nova.sms.ingest.complete` | Operation (update) | Set the ledger row's final state and verdict | low | UNVERIFIED |
| `nova.finance.extract` | Capability | Structured LLM extraction | medium | UNVERIFIED |
| `nova.finance.extract.transaction` | Operation (read, LLM) | Same transaction schema as the email logger | medium | UNVERIFIED |
| `nova.finance.hub` | Capability | Accounts, duplicates, router | high | UNVERIFIED |
| `nova.finance.hub.match_account` | Operation (read) | Account match from hint + sender ID; currency-gated | medium | UNVERIFIED |
| `nova.finance.hub.find_duplicate` | Operation (read) | Manual entry ±3 days, or same reference from any source | low | UNVERIFIED |
| `nova.finance.hub.route_transaction` | Operation (create) | Atomic route with `source_type = sms` | high | UNVERIFIED |
| `nova.notify.owner` | Capability | Owner channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Review nudge and security alerts | low | UNVERIFIED |

Dependencies declared elsewhere (not re-declared here): `nova.finance.hub_sync` (push to the external app) and `nova.finance.review` (approve/reject), both in `nova.finance.log-transactions-from-email`. Without them, records stay `pending` and review items cannot be resolved.

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.sms.ingest.receive.json` — `application/json` body ≤ 8 KB. Default.
- `nova.sms.ingest.receive.form` — `application/x-www-form-urlencoded` body ≤ 8 KB, for forwarders that cannot send JSON. Same normalised output and the same authentication; selected by the request content type, so no side-effect difference.
- `nova.finance.extract.transaction.default` / `.alternate` — temperature 0; the alternate is chosen by operator configuration only, never as a runtime fallback.
- `nova.finance.hub.route_transaction.default` — the same server-enforced gate as the email logger.

**Bindings:** `binding://nova/sms/webhook`, `binding://nova/sms/ingest-ledger`, `binding://nova/finance/extract-transaction`, `binding://nova/finance/match-account`, `binding://nova/finance/find-duplicate`, `binding://nova/finance/route-transaction`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: an automation-hub webhook with header authentication → a normalising code step → the LLM → a database router mirroring the email router. The webhook URL and token live only in the runtime (`secret_ref://sms-webhook-url`, `secret_ref://sms-forwarder-token`).

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Forwarded SMS (request) | `schema://nova/log-transactions-from-sms/forwarded-sms@0.1.0` | the installation's device | read (text held in memory only) |
| Ingest ledger row | `schema://nova/log-transactions-from-sms/ingest-row@0.1.0` | the requesting user's tenant | create; update (`state`, `verdict`, `outcome_ref`) |
| Hub record / review item | as in `nova.finance.log-transactions-from-email` | the requesting user's tenant | create |
| Account | `account-ref@0.1.0` | the requesting user | read |

**Authorization requirements (declared, never a grant):** `sms:ingest` (inbound, bound to the device token), `llm:extract`, `finance:hub:accounts:read`, `finance:hub:records:read`, `finance:hub:records:write`, `finance:review:write`, `notify:owner`. Secrets: `secret_ref://sms-forwarder-token`, `secret_ref://llm`, `secret_ref://finance-db-role`.

**Policy / trust:** risk class `high` (public inbound endpoint; financial writes). Message text is untrusted, and the endpoint is authenticated and rate-limited. One-time codes and non-financial texts never reach the LLM.

## C. Intent and applicability

- **User problem:** Some banks send alerts only by SMS, so those transactions never reach the email logger and must be typed by hand.
- **Desired outcome:** Each forwarded bank SMS becomes exactly one hub record or review item, without exposing personal texts or one-time codes.
- **Trigger or entry conditions:** An authenticated POST from the user's forwarder app.
- **Applicable situations:** Debit/credit alerts, card purchases, ATM withdrawals, transfers, refunds to a tracked account.
- **Out-of-scope situations:** Reading the phone's SMS inbox directly; personal messages; one-time codes; promotional SMS; replying to SMS; moving money; numbers or senders not on the allowlist (when one is configured).
- **Required inputs:** `forwarded_sms` (request), `timezone`, `autolog_scope` (see Section D).
- **Prerequisites:** The forwarder is installed on the user's own phone and configured with the endpoint URL and token, ideally filtered to bank sender IDs; accounts have match hints; the email logger's sync and review capabilities are installed.
- **Required capabilities:** see Section B (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port; the phone-side forwarder is outside the platform.
- **Assumptions:** Verified in the reference implementation: payload normalisation across forwarder formats, the hashed idempotency key, the shared router, one-time-code skipping and manual-duplicate reconcile. Not every control described here has evidence yet (Section P lists the tests still needed).
- **Ambiguity policy:** Use a documented safe default (review queue). Never guess an account, amount, currency or direction.
- **Vedaxi task type:** none.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| auth header | string (header name configurable, default `X-Forwarder-Token`) | yes | forwarder; secret | — | 401; no processing. |
| `forwarded_sms` | JSON or form body ≤ 8 KB | yes | forwarder; personal, financial | — | 400 on unparseable; 413 if too large. |
| `timezone` | IANA zone | yes | profile | — | 503 + `blocked` run record. |
| `sender_allowlist` | array of sender IDs/patterns | no | user | empty (keyword filter only) | — |
| `autolog_scope` | as in the email logger | yes (may be review-only) | approval record | review-only | Expired → review-only. |
| `max_age_days` | integer 1–60 | no | operator | 30 | Older messages → verdict `stale`, not logged. |

```json
{"$id":"schema://nova/log-transactions-from-sms/forwarded-sms@0.1.0","type":"object",
 "anyOf":[{"required":["text"]},{"required":["message"]},{"required":["body"]}],
 "properties":{"text":{"type":"string","maxLength":2000},"message":{"type":"string","maxLength":2000},"body":{"type":"string","maxLength":2000},
 "from":{"type":"string","maxLength":40},"sender":{"type":"string","maxLength":40},"address":{"type":"string","maxLength":40},
 "receivedStamp":{"type":["integer","string"]},"timestamp":{"type":["integer","string"]},"receivedAt":{"type":"string","format":"date-time"},
 "device_id":{"type":"string","maxLength":64}},"additionalProperties":true}
```

```json
{"$id":"schema://nova/log-transactions-from-sms/normalised-sms@0.1.0","type":"object","required":["sender","text","idempotency_key"],
 "properties":{"sender":{"type":"string","minLength":2,"maxLength":40},"text":{"type":"string","minLength":5,"maxLength":2000},
 "received_at":{"type":["string","null"],"format":"date-time"},"local_date":{"type":["string","null"],"format":"date"},
 "idempotency_key":{"type":"string","pattern":"^[a-f0-9]{64}$"},"device_id":{"type":["string","null"]}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| HTTP response | `schema://nova/log-transactions-from-sms/ack@0.1.0` | `accepted`, `duplicate`, `outcome` | the forwarder | no financial data in the body | status code matches the ledger state |
| Ingest ledger row | `ingest-row@0.1.0` | see schema | ingest ledger | financial metadata; no text | read-back |
| Hub record / review item | email-logger schemas, `source_type = sms` | — | finance hub | financial | read-back |

```json
{"$id":"schema://nova/log-transactions-from-sms/ack@0.1.0","type":"object","required":["accepted","duplicate","outcome"],
 "properties":{"accepted":{"type":"boolean"},"duplicate":{"type":"boolean"},
 "outcome":{"enum":["auto_logged","review","reconciled","skipped","duplicate","retry_later","rejected"]},"ingest_id":{"type":["string","null"]}}}
```

```json
{"$id":"schema://nova/log-transactions-from-sms/ingest-row@0.1.0","type":"object","required":["ingest_id","idempotency_key","sender","state"],
 "properties":{"ingest_id":{"type":"string"},"idempotency_key":{"type":"string","pattern":"^[a-f0-9]{64}$"},"sender":{"type":"string"},
 "received_at":{"type":["string","null"],"format":"date-time"},"state":{"enum":["processing","retryable","done"]},
 "verdict":{"enum":["auto_logged","review","reconciled","not_allowlisted","no_money_keywords","one_time_code","not_transaction","promotional","store_credit","amount_unreadable","stale",null]},
 "outcome_ref":{"type":["string","null"]},"attempts":{"type":"integer","minimum":0},"updated_at":{"type":"string","format":"date-time"}}}
```

`idempotency_key` = SHA-256 hex of `lower(sender) + "\n" + whitespace-collapsed text + "\n" + received_at truncated to the minute (UTC)`. If the forwarder sends no timestamp, the time part is empty, and this is disclosed in the ledger row.

Reused schemas (acknowledged): `extraction`, `hub-record`, `review-item`, `account-hint`, `account-ref`, `dup-query`, `dup-result`, `route-input` and `route-result` are defined in Section D of `nova.finance.log-transactions-from-email` (folder `15-log-transactions-from-email`) and used here unchanged. Own supporting schema:

```json
{"$id":"schema://nova/log-transactions-from-sms/delivery@0.1.0","type":"object","required":["delivered"],"properties":{"delivered":{"type":"boolean"},"message_id":{"type":["string","null"]},"kind":{"enum":["review_nudge","security_notice"]},"error":{"type":["string","null"]}}}
```

**Example (fictional)**

Request body: `{"from":"EXMPLBNK","text":"Example Bank: USD 18.75 debited from a/c ..0142 at Example Cafe on 03-11-26. Ref 551902. Not you? Call +1-555-0123.","receivedAt":"2026-11-03T14:02:00Z"}`

Response: `{"accepted":true,"duplicate":false,"outcome":"auto_logged","ingest_id":"ing_0057"}`. Hub record: `{"account_ref":"acct_ex_card_01","direction":"expense","amount":"18.75","currency":"USD","txn_date":"2026-11-03","counterparty":"Example Cafe","source_type":"sms","source_ref":"ref:551902","sync_status":"pending"}`.

## E. Constraints and budgets

- **Hard constraints:** reject requests without a valid token (constant-time comparison); never send one-time codes, personal texts or non-allowlisted senders to the LLM; never store SMS text after processing (the ledger keeps the key, sender, time and verdict only); never move money or reply to SMS; the auto-log gate is identical to the email logger's (high + high, opted-in account, ≤ cap, message newer than `enabled_at`); amounts are decimal strings with an explicit currency; the duplicate check (±3 days manual, same reference from any source) runs before logging.
- **Soft constraints:** respond within 25 seconds; prefer the forwarder's timestamp over arrival time.
- **Resource and tenant scope:** one installation token per user device; the requesting user's hub only.
- **Data sources allowed:** the forwarded message; hub accounts and records.
- **External transfers permitted:** filtered money SMS text to the approved LLM provider.
- **Time limit:** 25 seconds per request.
- **Cost or token budget:** at most one LLM call per unique money SMS; daily cap `TBD` (operator), after which items are stored as `retryable` and processed later.
- **Maximum steps:** 10 (S1–S10).
- **Maximum retries and correction cycles:** LLM 1 + 1 re-ask; router 2 (idempotent); a forwarder retry after `retryable` is processed again, up to 5 attempts, then the row goes to review.
- **Rate limits and concurrency:** 60 requests per minute per token; 10 failed authentications in 10 minutes → block for 30 minutes and notify the user.
- **Retention and logging constraints:** request bodies are never logged; ledger rows are kept per policy (`TBD`).
- **Prohibited actions:** accepting tokens in the query string; echoing message text in responses or errors; processing a message older than `max_age_days` into the hub.
- **Stop conditions:** token invalid; kill switch; approval expired (review-only).

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Ingest ledger row | authenticated, valid payload | Allowed | unique key |
| Create | Hub record (`source_type = sms`) | auto-log gate holds | Allowed within the standing approval | read-back |
| Create | Review item | other valid transactions | Allowed | row exists |
| Read | Accounts, records, ledger | own tenant | Allowed | — |
| Update | Ledger `state`, `verdict`, `outcome_ref`, `attempts` | this request owns the row | Allowed | read-back |
| Delete | ledger rows, records, review items | — | Not Allowed | — |
| External side effect | HTTP response to the forwarder | — | Allowed | — |
| External side effect | security alert or review nudge to the user | thresholds | Allowed | receipt |

- **Idempotency:** the ledger key is unique. Repeats of a `done` row → 200 `duplicate`; a `processing` row younger than 2 minutes → 200 `duplicate` (in progress); a `retryable` row → processed again. The router is also unique on `(source_type, source_ref)`.
- **Expected side effects:** ledger row; record or review item; optional notices.
- **Compensation:** user-approved reversal in the hub (as in the email logger).
- **Irreversible:** text sent to the LLM.
- **Ambiguous outcomes:** if the router times out, read back by `source_ref` before retrying; if the response to the forwarder is lost, the forwarder retries and gets `duplicate`.

## G. Autonomy and approval policy

- **Default autonomy level:** A1 — supervised execution, with the same narrowly bounded A2 auto-log exception as the email logger.
- **Planning level:** A2.
- **Execution level:** A2 inside `autolog_scope`; A0 for review items.
- **Verification level:** A2.
- **Recovery level:** A2.
- **Approval required for:** issuing or rotating the forwarder token; granting `autolog_scope`; every review item.
- **Actions never permitted:** processing unauthenticated requests; sending OTPs or personal texts to the LLM; storing SMS text; replying to senders; moving money.
- **Conditions requiring clarification:** a currency different from the matched account's currency; two possible accounts.
- **Conditions requiring escalation:** an authentication-failure burst; a forwarder sending many non-allowlisted senders (it may be forwarding everything); repeated `retryable` rows.
- **Safe state after stop, cancellation or escalation:** ledger rows `retryable` or `done`; no partial routes.

## H. Execution plan

**S1 — Authenticate the request**
1. Objective: accept only the user's own forwarder.
2. Capability: `nova.sms.ingest.receive` (authentication part).
3. Input mapping: auth header → compared with `secret_ref://sms-forwarder-token`.
4. Dependencies: none.
5. Preconditions: HTTPS; kill switch off; token not blocked.
6. Action / side effects: constant-time comparison; failed attempts counted per source.
7. Autonomy: A2.
8. Timeout / retry: 1 s / none.
9. Idempotency: not applicable.
10. Postconditions: an authenticated request bound to one installation.
11. Verification: token matches; the installation is active.
12. Failure routes: invalid or missing → 401 with an empty body; burst threshold → block and S10 security notice.
13. Escalation triggers: authentication burst.
14. Evidence: auth result and installation ID (never the token).

**S2 — Validate and normalise**
1. Objective: one canonical message.
2. Capability: `nova.sms.ingest.receive.json` or `.form`.
3. Input mapping: body → `normalised-sms@0.1.0` (text from `text|message|body`; sender from `from|sender|address`; time from `receivedAt|receivedStamp|timestamp`, epoch seconds or milliseconds, or ISO).
4. Dependencies: S1.
5. Preconditions: body ≤ 8 KB.
6. Action / side effects: parse, trim, collapse whitespace, convert the time to UTC and to the user's local date; compute the idempotency key.
7. Autonomy: A2.
8. Timeout / retry: 1 s / none.
9. Idempotency: deterministic key.
10. Postconditions: a valid normalised message.
11. Verification: schema valid; time not more than 5 minutes in the future.
12. Failure routes: unparseable → 400; too large → 413; future time → 400; older than `max_age_days` → verdict `stale` (logged in the ledger, 200).
13. Escalation: none.
14. Evidence: content type, key, sender, time.

**S3 — Privacy pre-filter**
1. Objective: only money messages go further.
2. Capability: deterministic filter.
3. Input mapping: sender, text, `sender_allowlist`.
4. Dependencies: S2.
5. Preconditions: none.
6. Action: allowlist configured and sender not on it → `not_allowlisted`; one-time-code patterns ("OTP", "verification code", "do not share", code-only messages) → `one_time_code`; no money keyword or amount pattern → `no_money_keywords`. Each of these ends with a ledger verdict and 200, and **no LLM call**.
7. Autonomy: A2.
8. Timeout / retry: 1 s / none.
9. Idempotency: pure.
10. Postconditions: pass or a filter verdict.
11. Verification: filtered messages never reach S5 (asserted in tests).
12. Failure routes: none.
13. Escalation: more than 50 % of messages from one installation fail the allowlist over a day → suggest filtering on the phone.
14. Evidence: filter verdict.

**S4 — Register the idempotency key**
1. Objective: process each SMS once.
2. Capability: `nova.sms.ingest.register`.
3. Input mapping: key, sender, `received_at`.
4. Dependencies: S3 (filtered messages are registered with their verdict directly).
5. Preconditions: none.
6. Action / side effects: insert the row with `processing`, or read the existing one.
7. Autonomy: A2.
8. Timeout / retry: 5 s / 2.
9. Idempotency: unique key.
10. Postconditions: this request owns a `processing` row, or a duplicate outcome is known.
11. Verification: the returned row state.
12. Failure routes: `done` → 200 `duplicate`; `processing` < 2 min → 200 `duplicate`; `retryable` → take ownership (`attempts + 1`); store down → 503 `retry_later`.
13. Escalation: none.
14. Evidence: ingest ID, attempt number.

**S5 — Extract the transaction**
1. Objective: structured fields with confidence.
2. Capability: `nova.finance.extract.transaction`.
3. Input mapping: text, sender, local date, account names.
4. Dependencies: S4.
5. Preconditions: approved provider; daily cap not reached.
6. Action / side effects: one LLM call.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 1 + 1 re-ask.
9. Idempotency: no state change.
10. Postconditions: `extraction@0.1.0` (email-logger schema).
11. Verification: schema valid; currency present whenever amount present.
12. Failure routes: provider down or cap reached → ledger `retryable`, respond 503 `retry_later`.
13. Escalation: repeated `retryable` → notify the user.
14. Evidence: model ID, valid flag.

**S6 — Check fields and skip rules**
1. Objective: drop hallucinations and non-transactions.
2. Capability: deterministic checks.
3. Input mapping: extraction + text.
4. Dependencies: S5.
5. Preconditions: valid extraction.
6. Action: the amount, last-4 and reference must appear in the text; skip `not_transaction`, `promotional`, `store_credit`; unreadable amount → `amount_unreadable`.
7. Autonomy: A2.
8. Timeout / retry: 1 s / none.
9. Idempotency: pure.
10. Postconditions: a supported transaction or a skip verdict.
11. Verification: support map complete.
12. Failure routes: unsupported account hint → null (routes to review).
13. Escalation: none.
14. Evidence: support map.

**S7 — Match the account**
1. Objective: one account or an honest "not sure".
2. Capability: `nova.finance.hub.match_account.default`.
3. Input mapping: `account_hint` + sender ID + currency.
4. Dependencies: S6.
5. Preconditions: accounts with hints (sender IDs can be hints).
6. Action / side effects: read-only.
7. Autonomy: A2.
8. Timeout / retry: 5 s / 2.
9. Idempotency: read-only.
10. Postconditions: `{account_ref|null, match}`.
11. Verification: `high` is unique and currency-equal.
12. Failure routes: ambiguity → `medium`.
13. Escalation: none.
14. Evidence: match level.

**S8 — Check for duplicates**
1. Objective: no double counting across manual entries, email and SMS.
2. Capability: `nova.finance.hub.find_duplicate`.
3. Input mapping: account, direction, amount, currency, date, reference.
4. Dependencies: S7.
5. Preconditions: amount and date known.
6. Action / side effects: read-only. Look for a manual entry with the same account, direction and exact amount within ±3 days, or any record with the same normalised reference.
7. Autonomy: A2.
8. Timeout / retry: 5 s / 2.
9. Idempotency: read-only.
10. Postconditions: `none`, `exact` or `multiple`.
11. Verification: the match belongs to the tenant.
12. Failure routes: `multiple` → review `possible_duplicate`.
13. Escalation: none.
14. Evidence: matched record ID.

**S9 — Route atomically**
1. Objective: log, queue or reconcile, exactly once.
2. Capability: `nova.finance.hub.route_transaction.default` (`source_type = sms`; `source_ref` = `ref:<normalised reference>` if present, else `sms:<key>`).
3. Input mapping: S6–S8 results, effective scope.
4. Dependencies: S8.
5. Preconditions: write scopes.
6. Action / side effects: insert a `pending` record, a review item, or nothing (reconciled), with the gate recomputed server-side.
7. Autonomy: A2 inside scope.
8. Timeout / retry: 10 s / 2 after read-back.
9. Idempotency: unique `(source_type, source_ref)`.
10. Postconditions: one outcome stored.
11. Verification: outcome = read-back.
12. Failure routes: conflict → re-run S8–S9 once → else review; store down → `retryable`, 503.
13. Escalation: none.
14. Evidence: record or review ID.

**S10 — Complete, respond and notify**
1. Objective: close the ledger row and answer the forwarder.
2. Capability: `nova.sms.ingest.complete`; `nova.notify.owner.send` (optional).
3. Input mapping: outcome → ledger row and `ack@0.1.0`.
4. Dependencies: S9 (or the earlier exits).
5. Preconditions: none.
6. Action / side effects: set the row to `done` with its verdict; respond 200 with the ack (no amounts or text); review nudges are batched at most hourly; security notices are sent immediately.
7. Autonomy: A2.
8. Timeout / retry: 5 s / 2 (ledger).
9. Idempotency: ledger key; nudge key = hour bucket.
10. Postconditions: row `done` or `retryable`; response sent.
11. Verification: the HTTP status matches the row state (200 ↔ done, 503 ↔ retryable).
12. Failure routes: ledger update fails → respond 503 so the forwarder retries (the router is idempotent).
13. Escalation: none.
14. Evidence: ingest ID, status code.

Records created here are mirrored to the external app by `nova.finance.hub_sync.push_batch` (email-logger recipe), which runs on its own schedule.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Authentication | no unauthenticated request reaches S2 | endpoint log | 100 % | block; security notice |
| Privacy filter | no OTP, non-allowlisted or keyword-less message reaches the LLM | filter log vs LLM call log | zero | stop LLM calls; escalate |
| No text persistence | no SMS text in ledger, logs or responses | storage scan in tests | none | escalate |
| Idempotency | one ledger row per key; one record per `source_ref` | unique constraints | holds | report defect |
| Gate | auto-logged records satisfy every gate condition | read-back vs approval | 100 % | move to review |
| Duplicates | manual ±3 days and same-reference duplicates reconciled | S8 log | holds | review |
| Status truth | 200 only when the row is `done`; 503 only when `retryable` | response log vs ledger | consistent | fix response mapping |
| Output schema | ack and ledger rows valid | validator | valid | fix; never fake outcomes |

A 200 to the forwarder means the message is safely recorded (done). It does not mean the transaction was logged; the ack's `outcome` says what happened.

## J. Self-correction and recovery

- **Failure classes:** authentication, payload validation, staleness, filter rejection, ledger conflict, LLM failure or cap, unsupported field, account or duplicate ambiguity, router conflict, store outage.
- **Permitted corrections:** mark `retryable` and let the forwarder retry; re-ask the LLM once; downgrade to review; re-run S8–S9 once.
- **Forbidden corrections:** accepting a request without a valid token; skipping the privacy filter; sending a one-time code to the LLM "to be sure"; storing text for later; lowering the gate.
- **Maximum correction cycles:** 1 per request; 5 forwarder attempts per key, then review.
- **Reverification required after correction:** yes — S6–S8.
- **Stop conditions:** token blocked; kill switch; budget exhausted.
- **Recovery or compensation:** user-approved reversal in the hub.
- **What must be reported:** security events, retryable backlog, review reasons.

## K. Independent challenge

- **Assumptions to challenge:** the SMS really came from the bank (sender IDs can be spoofed — an SMS that asks the user to act is never logged as a transaction without an amount, account and reference); the same purchase has not already arrived by email.
- **Alternative explanation:** a "debited" SMS for a pending authorisation later reversed; a failed transaction notice.
- **Contradictory evidence:** a later reversal SMS with the same reference (route to review as a possible reversal).
- **Could mislead:** a silent forwarder (phone off, app killed) makes capture look complete. The run summary shows the last message time per installation and flags more than 72 hours of silence.
- **Separate verifier:** the server-side gate and the ledger-vs-response consistency check are independent of the extraction.
- **Findings requiring disclosure:** spoofing suspicion, possible reversals, forwarder silence.

## L. Human escalation

- **Triggers:** authentication bursts; forwarder silent > 72 hours (if it was active before); repeated `retryable`; possible spoofed or reversal messages; review items.
- **Why the agent cannot decide:** device security and genuine ambiguity in money records need the user.
- **Information presented:** installation, counts, last message time, the review item fields (no full text).
- **Decision requested:** rotate the token; check the phone app; approve or reject review items.
- **Options and consequences:** rotating invalidates the old token immediately; ignoring leaves capture degraded.
- **Safe behaviour while waiting:** keep rejecting bad tokens; keep `retryable` rows.
- **If no response:** blocks expire after 30 minutes; review items stay pending.
- **Resume:** the new token is configured on the phone; the next message is processed normally.

## M. Completion conditions

Mandatory (per request): authentication passed; the privacy filter held; the ledger row is `done` or `retryable` and consistent with the response; any record satisfies the gate; no text persisted; evidence references present.

- `completed` — the message reached a final verdict (logged, review, reconciled or skipped) and the row is `done`.
- `partially_completed` — routed, but the review nudge or security notice could not be delivered.
- `blocked` — authentication failed, the payload was invalid, or there was no time zone or approval configuration.
- `failed` — the store or LLM was unavailable, so the row is `retryable` (503 returned).
- `cancelled` — kill switch, or the token was revoked during processing.

## N. Final output contract

Return (per request, in the run record): run ID; recipe ID and version; status; one-line outcome ("logged 18.75 USD expense" in the record, never in the HTTP body); ack; ingest ID; step statuses S1–S10; verification results; evidence references (ledger row, record or review ID); approvals and scope (approval ID); side effects and reconciliation status; corrections and attempts; unresolved challenges; limitations; escalation state; learning candidate. Never include the SMS text, the token or full account numbers.

## O. Learning and revision

- **Outcome metrics:** filter rejection rate by reason, LLM calls per day, auto-log precision (user reversals), review rate, duplicate catches (manual and cross-channel), retryable rate, authentication failures.
- **Evidence retained:** ledger rows and run records per policy.
- **Data minimisation:** no SMS text anywhere after processing.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}` — for example a new one-time-code phrasing.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests; filter changes also need the OTP fixture set.
- **Promotion approval:** publisher review plus Skill Port review.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (fixture bank SMS from a test forwarder) | one record, ack `auto_logged` | UNVERIFIED on Skill Port; observed in the reference implementation with test messages |
| Invalid input (unparseable body) | 400; nothing stored | TBD |
| Missing input (no text field) | 400 | TBD |
| Form-encoded payload | normalised identically | TBD |
| Missing capability (router binding absent) | 503 `retry_later`; row `retryable` | TBD |
| Permission denied (missing or invalid token) | 401; no processing | TBD |
| Unauthorized scope (token of another installation) | rejected | TBD |
| Tenant isolation | routing never touches another tenant | TBD |
| One-time code | verdict `one_time_code`; zero LLM calls | TBD |
| Non-allowlisted sender | `not_allowlisted`; zero LLM calls | TBD |
| Stale message (> 30 days) | `stale`; nothing logged | TBD |
| Manual entry ±3 days | `reconciled` | TBD |
| Same reference already logged from email | `reconciled` | TBD |
| Timeout (router) | read-back; no duplicate | TBD |
| Duplicate / replay (forwarder sends twice) | second ack `duplicate` | TBD |
| False success (200 returned while row not done) | consistency check fails the test | TBD |
| Failed verification (gate audit) | record moved to review | TBD |
| Correction budget exhausted (5 attempts) | row to review | TBD |
| Escalation (auth burst) | token blocked 30 min; user notified | TBD |
| Cancellation (token revoked mid-request) | `cancelled`; nothing logged | TBD |
| Compensation (user reversal) | offsetting entry only with approval | TBD |
| Output schema and status | valid ack and ledger row | TBD |
| Compatibility / regression (new forwarder app format) | normaliser fixture passes | TBD |

All tests use a test forwarder or recorded fictional payloads, sandbox accounts and rolled-back transactions; never real SMS or real accounts. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi inbound-webhook support, token issuance UI and binding IDs — TBD (owner: publisher, with Skill Port).
- Forwarder apps differ in field names and retry behaviour; the normaliser covers common ones, but new apps need fixtures (owner: publisher).
- Sender IDs are not authenticated by the phone network in all countries; spoofed bank SMS are possible.
- Messages without a timestamp produce weaker idempotency keys (two identical texts on different days collide); this is disclosed in the ledger row.
- The daily LLM cap default — TBD (owner: operator).
- The phone must stay online with the forwarder running; offline periods delay capture.
