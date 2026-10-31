# Upcoming payments radar

## A. Identity and ownership

- **Recipe ID:** `nova.finance.upcoming-payments-radar` (proposed until assigned by Skill Port)
- **Name:** Upcoming payments radar
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** finance
- **Tags:** payments, renewals, bills, instalments, card-dues, email-extraction, payment-schedule, needs-review · Vedaxi task type: none (no supported task type covers this today)
- **Summary:** On a schedule, the agent scans the user's connected mailboxes for messages about future money obligations (subscription renewals, utility and service bills, instalments/EMIs, card dues), extracts each one with an LLM into a confidence-scored record, and keeps a deduplicated payment schedule. Anything uncertain or incomplete goes to needs-review instead of being guessed.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** A mailbox API with metadata listing and on-demand full-message reads; a structured-output LLM; a database that exposes the schedule only through audited dispatcher functions; the finance hub's account list (read-only, for a suggested paying account). All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B). Downstream recipes: `nova.finance.payment-reminders` (alerts) and `nova.finance.auto-mark-bills-paid` (receipts).
- **Evidence status:** Partially verified — detection, extraction, deduplication, roll-forward and cancellation handling have run end-to-end in Nova's reference implementation (catalogue status: Built). Not every control described here has evidence yet (Section P lists the tests still needed). Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.mail.candidates` | Capability | Select candidate messages from connected mailboxes (metadata only) | medium | UNVERIFIED |
| `nova.mail.candidates.list` | Operation (read) | Return not-yet-judged messages that match the billing selector, deduplicated by RFC Message-ID | medium | UNVERIFIED |
| `nova.mail.body` | Capability | Read one message body on demand without persisting it | medium | UNVERIFIED |
| `nova.mail.body.fetch_transient` | Operation (read) | Return decoded plain text (≤ 15,000 characters) for one message; never stored | medium | UNVERIFIED |
| `nova.mail.verdicts` | Capability | Remember which messages a pipeline has definitively judged | low | UNVERIFIED |
| `nova.mail.verdicts.mark` | Operation (create) | Record `(message_ref, pipeline, verdict)` once a definitive verdict exists | low | UNVERIFIED |
| `nova.finance.extract` | Capability | Structured LLM extraction from untrusted message text | medium | UNVERIFIED |
| `nova.finance.extract.obligation` | Operation (read, LLM) | Classify a message and extract one obligation with per-field confidence | medium | UNVERIFIED |
| `nova.finance.obligations` | Capability | The user's payment schedule | medium | UNVERIFIED |
| `nova.finance.obligations.apply` | Operation (create + update) | Route one extraction: skip, needs-review, upsert, or dismiss on cancellation — atomically | medium | UNVERIFIED |
| `nova.finance.obligations.list` | Operation (read) | Read schedule entries for verification and reporting | low | UNVERIFIED |
| `nova.finance.hub` | Capability | The finance hub's accounts and records | medium | UNVERIFIED |
| `nova.finance.hub.match_account` | Operation (read) | Suggest a paying account from hints, gated on currency | low | UNVERIFIED |
| `nova.notify.owner` | Capability | Deliver a message to the requesting user's own channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Send the new needs-review notice | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.mail.candidates.list.default` — selector over the synced metadata store: a sender on the configured biller list passes; any other sender needs a forward-looking billing token in the subject (renew, renewal, due, invoice, bill, expiry, instalment, auto-pay). Excludes the user's own outbound domains, transaction-debit subjects (handled by `nova.finance.log-transactions-from-email`), shipping, one-time-code and newsletter noise. Read-only.
- `nova.mail.body.fetch_transient.default` — mailbox API full-format read, MIME walk, base64url decode, HTML stripped, truncated to 15,000 characters. The automation hub saves no execution data for this flow, so bodies never persist.
- `nova.finance.extract.obligation.default` — primary structured-output LLM, temperature 0, strict JSON, about 700 output tokens.
- `nova.finance.extract.obligation.alternate` — a second, operator-configured LLM provider with the same schema and data-processing terms. It is chosen **by operator configuration only**. It is never a runtime fallback, because switching providers changes where message text is sent.
- `nova.finance.obligations.apply.default` — one security-definer database function with a pinned search path, called through the automation hub. Atomic per message.
- `nova.finance.hub.match_account.default` — matches institution names and last-4 hints against the user's accounts; returns `high | medium | low | none`. A match is only offered when the account currency equals the obligation currency.
- `nova.notify.owner.send.default` — the user's own chat channel (for example Telegram), plain text.

**Bindings:** `binding://nova/mail/candidates`, `binding://nova/mail/body-transient`, `binding://nova/mail/verdicts`, `binding://nova/finance/extract-obligation`, `binding://nova/finance/obligations-apply`, `binding://nova/finance/obligations-list`, `binding://nova/finance/match-account`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: a scheduled automation-hub workflow (every 30 minutes) calls the mailbox API and the LLM. The schedule lives in a PostgreSQL schema with row-level security, reachable only through `SECURITY DEFINER` dispatcher functions.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Message metadata | `schema://nova/upcoming-payments-radar/message-ref@0.1.0` | the requesting user's connected mailboxes | read |
| Message body | transient text only, no schema stored | one message, one step | read (never stored) |
| Verdict | `schema://nova/upcoming-payments-radar/verdict@0.1.0` | the requesting user | create |
| Obligation | `schema://nova/upcoming-payments-radar/obligation@0.1.0` | the requesting user's tenant | create, read, update (listed fields) |
| Account (finance hub) | `schema://nova/upcoming-payments-radar/account-ref@0.1.0` | the requesting user | read |

**Authorization requirements (declared, never a grant):** `mail:metadata:read`, `mail:body:read_transient`, `mail:verdicts:write`, `llm:extract`, `finance:obligations:read`, `finance:obligations:write`, `finance:hub:accounts:read`, `notify:owner`. Mailbox OAuth credentials: `secret_ref://mail-oauth`; LLM key: `secret_ref://llm`; database role: `secret_ref://finance-db-role`.

**Policy / trust:** risk class `medium` (reads private correspondence; writes reminder records only; never moves money). Message text is untrusted input: instructions inside an email cannot change routing, scopes or outputs.

## C. Intent and applicability

- **User problem:** Renewal notices, bills and card-due reminders are spread across several inboxes, so payments get missed or surprise the user.
- **Desired outcome:** One accurate, deduplicated schedule of upcoming obligations. Each entry has a source message and a confidence level, and uncertain items are clearly marked.
- **Trigger or entry conditions:** A scheduled run (default every 30 minutes) after the user has enabled the radar for specific mailboxes, or an on-demand "scan for upcoming payments now".
- **Applicable situations:** Subscription renewals, utility/telecom/insurance bills, loan instalments (EMIs), credit-card statement dues, invoices with a due date, trial-to-paid conversions.
- **Out-of-scope situations:** Paying, scheduling a payment or contacting a biller; marking bills paid (`nova.finance.auto-mark-bills-paid`); logging transactions (`nova.finance.log-transactions-from-email`); sending reminders (`nova.finance.payment-reminders`); money the user will *receive*; reading mailboxes the user has not connected.
- **Required inputs:** `mailbox_accounts`, `timezone` (see Section D).
- **Prerequisites:** Mailbox metadata sync running; mailbox OAuth consent granted for read access; an LLM provider approved by the operator for processing email text; the obligation store installed.
- **Required capabilities:** `nova.mail.candidates`, `nova.mail.body`, `nova.mail.verdicts`, `nova.finance.extract`, `nova.finance.obligations`, `nova.finance.hub` (all proposed, UNVERIFIED); `nova.notify.owner` (optional).
- **Unsupported or missing capabilities:** None registered on Skill Port; every binding is a dependency to resolve before activation.
- **Assumptions:** Verified in the reference implementation: the selector, transient body fetch, LLM classification (including phishing and cancellation), roll-forward of recurring dates, deduplication by `uniq_key`, and the earliest-future-cycle rule. Not every control described here has evidence yet (Section P lists the tests still needed).
- **Ambiguity policy:** Use a documented safe default. Never guess a date, amount or currency: missing date → `needs_date`; low or medium confidence, an ambiguous currency glyph or a conflicting extraction → `needs_review`.
- **Vedaxi task type:** none.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `mailbox_accounts` | array of mailbox installation refs, 1–10 | yes | installation config; personal | — | `blocked` with "no mailbox connected". |
| `timezone` | IANA zone string | yes | user profile | — | `blocked`; dates cannot be compared without it. |
| `batch_size` | integer 1–20 | no | operator | 8 | Clamp to range; report. |
| `biller_senders` | array of sender domain patterns | no | user/operator | empty | Empty: subject tokens only. |
| `billing_subject_tokens` | array of strings | no | operator | built-in token list | Empty: built-in list. |
| `own_domains` | array of domains | no | user | empty | Empty: no outbound exclusion; report as a limitation. |
| `backfill_cutoff_days` | integer 0–90 | no | operator | 10 | Messages older than this at first run get the verdict `backfill` and are not extracted. |
| `auto_schedule_min_confidence` | enum `high`, `medium` | no | operator | `high` | Anything below → `needs_review`. |

```json
{"$id":"schema://nova/upcoming-payments-radar/config@0.1.0","type":"object","required":["mailbox_accounts","timezone"],
 "properties":{"mailbox_accounts":{"type":"array","minItems":1,"maxItems":10,"items":{"type":"string"}},
 "timezone":{"type":"string","minLength":3},"batch_size":{"type":"integer","minimum":1,"maximum":20},
 "biller_senders":{"type":"array","items":{"type":"string"}},"billing_subject_tokens":{"type":"array","items":{"type":"string"}},
 "own_domains":{"type":"array","items":{"type":"string"}},"backfill_cutoff_days":{"type":"integer","minimum":0,"maximum":90},
 "auto_schedule_min_confidence":{"enum":["high","medium"]}}}
```

```json
{"$id":"schema://nova/upcoming-payments-radar/extraction@0.1.0","type":"object",
 "required":["class","direction","merchant","series_key","confidence"],
 "properties":{"class":{"enum":["alertable","informational","past_event","phishing","promotional","cancellation","payment_confirmation"]},
 "direction":{"enum":["pay","receive"]},"kind":{"enum":["subscription_renewal","utility_bill","instalment","card_due","invoice","trial_conversion","other"]},
 "merchant":{"type":"string","maxLength":120},"series_key":{"type":"string","pattern":"^[a-z0-9][a-z0-9-]{2,79}$"},
 "amount":{"type":["string","null"],"pattern":"^[0-9]{1,13}(\\.[0-9]{1,4})?$"},"currency":{"type":["string","null"],"pattern":"^[A-Z]{3}$"},
 "due_date":{"type":["string","null"],"format":"date"},"recurrence":{"enum":["weekly","monthly","quarterly","annual","one_off","unknown"]},
 "auto_charge":{"type":"boolean"},"external_ref":{"type":["string","null"],"maxLength":64},
 "confidence":{"enum":["high","medium","low"]},"field_confidence":{"type":"object","additionalProperties":{"enum":["high","medium","low"]}},
 "paid_date":{"type":["string","null"],"format":"date"},"account_hint":{"type":["string","null"],"maxLength":64}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `radar_run` | object `schema://nova/upcoming-payments-radar/radar-run@0.1.0` | `run_id`, `status`, `examined`, `outcomes` | run record; owner channel only for new needs-review items | financial, personal; owner only | counts reconcile with verdict rows and obligation read-back (Section I) |
| Obligation rows | `obligation@0.1.0` | see schema | obligation store | financial; owner only | read-back after apply |

```json
{"$id":"schema://nova/upcoming-payments-radar/obligation@0.1.0","type":"object",
 "required":["obligation_id","uniq_key","series_key","merchant","recurrence","status","extract_confidence","source_message_ref"],
 "properties":{"obligation_id":{"type":"string"},"uniq_key":{"type":"string","pattern":"^(sub|once):[a-z0-9-]+(:[A-Za-z0-9._-]+)?$"},
 "series_key":{"type":"string"},"merchant":{"type":"string"},"kind":{"type":"string"},
 "amount":{"type":["string","null"],"pattern":"^[0-9]{1,13}(\\.[0-9]{1,4})?$"},"currency":{"type":["string","null"],"pattern":"^[A-Z]{3}$"},
 "recurrence":{"enum":["weekly","monthly","quarterly","annual","one_off","unknown"]},"auto_charge":{"type":"boolean"},
 "due_date":{"type":["string","null"],"format":"date"},
 "status":{"enum":["upcoming","due_today","overdue","paid","dismissed","snoozed","needs_date","needs_review"]},
 "series_dismissed":{"type":"boolean"},"alert_state":{"type":"object","properties":{"new":{"type":"boolean"},"t3":{"type":"boolean"},"t1":{"type":"boolean"},"due":{"type":"boolean"},"overdue":{"type":"boolean"}}},
 "pay_account_ref":{"type":["string","null"]},"extract_confidence":{"enum":["high","medium","low"]},
 "external_ref":{"type":["string","null"]},"review_reason":{"type":["string","null"]},"source_message_ref":{"type":"string"}}}
```

```json
{"$id":"schema://nova/upcoming-payments-radar/radar-run@0.1.0","type":"object","required":["run_id","status","examined","outcomes"],
 "properties":{"run_id":{"type":"string"},"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},
 "examined":{"type":"integer","minimum":0},"outcomes":{"type":"object","properties":{"scheduled":{"type":"integer"},"updated":{"type":"integer"},
 "needs_review":{"type":"integer"},"needs_date":{"type":"integer"},"dismissed_by_cancellation":{"type":"integer"},"handed_to_paid_tracking":{"type":"integer"},
 "skipped_non_alertable":{"type":"integer"},"phishing":{"type":"integer"},"deferred":{"type":"integer"}}},
 "items":{"type":"array","items":{"$ref":"schema://nova/upcoming-payments-radar/obligation@0.1.0"}},"limitations":{"type":"array","items":{"type":"string"}}}}
```

Supporting schemas (one per line):

```json
{"$id":"schema://nova/upcoming-payments-radar/message-ref@0.1.0","type":"object","required":["message_ref","mailbox_ref","received_at"],"properties":{"message_ref":{"type":"string","description":"hash of the RFC Message-ID"},"mailbox_ref":{"type":"string"},"received_at":{"type":"string","format":"date-time"},"from_domain":{"type":"string"},"subject":{"type":"string","maxLength":200}}}
{"$id":"schema://nova/upcoming-payments-radar/body-text@0.1.0","type":"object","required":["message_ref","text"],"description":"transient, never persisted","properties":{"message_ref":{"type":"string"},"subject":{"type":"string","maxLength":200},"text":{"type":"string","minLength":1,"maxLength":15000}}}
{"$id":"schema://nova/upcoming-payments-radar/verdict@0.1.0","type":"object","required":["message_ref","pipeline","verdict","judged_at"],"properties":{"message_ref":{"type":"string"},"pipeline":{"const":"upcoming"},"verdict":{"enum":["alertable","informational","past_event","phishing","promotional","cancellation","payment_confirmation","needs_review","backfill"]},"judged_at":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/upcoming-payments-radar/account-hint@0.1.0","type":"object","properties":{"account_hint":{"type":["string","null"],"maxLength":64},"sender_domain":{"type":["string","null"]},"currency":{"type":["string","null"],"pattern":"^[A-Z]{3}$"}}}
{"$id":"schema://nova/upcoming-payments-radar/account-ref@0.1.0","type":"object","required":["match"],"properties":{"account_ref":{"type":["string","null"]},"display_name":{"type":["string","null"]},"currency":{"type":["string","null"],"pattern":"^[A-Z]{3}$"},"last4":{"type":["string","null"],"pattern":"^[0-9]{4}$"},"match":{"enum":["high","medium","low","none"]}}}
{"$id":"schema://nova/upcoming-payments-radar/obligation-query@0.1.0","type":"object","properties":{"obligation_ids":{"type":"array","items":{"type":"string"}},"uniq_keys":{"type":"array","items":{"type":"string"}},"statuses":{"type":"array","items":{"type":"string"}},"limit":{"type":"integer","minimum":1,"maximum":500}}}
{"$id":"schema://nova/upcoming-payments-radar/delivery@0.1.0","type":"object","required":["delivered"],"properties":{"delivered":{"type":"boolean"},"message_id":{"type":["string","null"]},"channel_ref":{"type":"string"},"error":{"type":["string","null"]}}}
```

**Example (fictional)**

Input: `{"mailbox_accounts":["mbx_ex_01","mbx_ex_02"],"timezone":"America/New_York","own_domains":["example.org"]}`

Output: `{"run_id":"run_0012","status":"completed","examined":4,"outcomes":{"scheduled":1,"updated":1,"needs_review":1,"needs_date":0,"dismissed_by_cancellation":0,"handed_to_paid_tracking":0,"skipped_non_alertable":1,"phishing":0,"deferred":0},"items":[{"obligation_id":"obl_0007","uniq_key":"sub:example-streaming-plus","series_key":"example-streaming-plus","merchant":"Example Streaming","kind":"subscription_renewal","amount":"12.99","currency":"USD","recurrence":"monthly","auto_charge":true,"due_date":"2026-11-12","status":"upcoming","series_dismissed":false,"alert_state":{"new":false,"t3":false,"t1":false,"due":false,"overdue":false},"pay_account_ref":"acct_ex_card_01","extract_confidence":"high","external_ref":null,"review_reason":null,"source_message_ref":"msg_ex_4f2a"}],"limitations":[]}`

## E. Constraints and budgets

- **Hard constraints:** never move money, schedule a payment, reply to, forward or click links in a biller's message; never persist message bodies; amounts are decimal strings and any amount carries an explicit ISO 4217 currency; never invent a due date, amount or currency; money the user receives (`direction = receive`) is never scheduled; phishing-classified messages never create an obligation; only mailboxes in `mailbox_accounts` are read; a dismissed series stays dismissed unless the user restores it.
- **Soft constraints:** prefer the earliest upcoming cycle; keep merchant names as the biller writes them; keep review reasons short.
- **Resource and tenant scope:** the requesting user's connected mailboxes, obligations and finance-hub account list only.
- **Data sources allowed:** synced message metadata, transient message bodies, the user's account list.
- **External transfers permitted:** message text (≤ 15,000 characters) to the operator-approved LLM provider, only to run extraction.
- **Time limit:** 10 minutes per run; 60 seconds per message.
- **Cost or token budget:** at most `batch_size` LLM calls per run (default 8); per-run token cap `TBD` (set by the operator).
- **Maximum steps:** 10 (S1–S10), repeated S3–S8 per message.
- **Maximum retries and correction cycles:** body fetch 2 retries; LLM 1 retry on transport error, 1 re-ask on schema failure; apply 2 retries (idempotent); correction cycles: 1 per message.
- **Rate limits and concurrency:** one concurrent run per user (run lock); mailbox API and LLM provider limits apply.
- **Retention and logging constraints:** obligations and verdicts kept until the user deletes them or per operator policy (`TBD`); logs hold message references and verdicts only, never body text, credentials or full account numbers.
- **Prohibited actions:** deleting or labelling mail; unsubscribing; changing finance-hub balances or records; contacting anyone but the user.
- **Stop conditions:** mailbox authorization missing or expired; LLM provider not approved or unavailable (items deferred); kill switch on; run lock held by another run.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Obligation (one per `uniq_key`) | extraction valid, class `alertable`, direction `pay` | Allowed (standing scope set when the user enables the radar) | read-back by `uniq_key` |
| Create | Verdict (one per message per pipeline) | definitive outcome reached | Allowed | row exists |
| Read | Message metadata, transient body, obligations, account list | own tenant, connected mailboxes only | Allowed | — |
| Update | Obligation: `due_date` (earlier future cycle only), `amount`, `currency`, `external_ref`, `pay_account_ref`, `extract_confidence`, `status` (to `needs_review`, `needs_date`, `upcoming`) | same `uniq_key`; never re-open a dismissed series | Allowed | read-back shows only the listed fields changed |
| Update | Obligation: `status = dismissed`, `series_dismissed = true` | cancellation message with an **exact** `series_key` match | Allowed (reported in the run summary; user can restore) | read-back |
| Update | Obligation dismissal on a fuzzy (containment) match | — | Not Allowed autonomously → creates a needs-review item | — |
| Delete | any obligation, verdict or message | — | Not Allowed | — |
| External side effect | text sent to the LLM provider | approved provider only | Allowed (standing scope) | provider call log (no body text) |
| External side effect | needs-review notice to the user | new needs-review items exist | Allowed | delivery receipt |

- **Idempotency:** the obligation identity is `uniq_key` (`sub:<series_key>` for recurring, `once:<series_key>:<external_ref or first 16 hex of SHA-256(message_ref)>` for one-off). The verdict primary key is `(message_ref, pipeline)`. Re-processing the same message is a no-op.
- **Expected side effects:** obligation rows created or updated; verdict rows written; one optional notice to the user.
- **Compensation:** the user can restore a dismissed series, edit fields, or dismiss an entry. The previous field values are kept in the obligation's change history (`TBD` table name), so a wrong update can be reverted.
- **Irreversible:** message text already sent to the LLM provider.
- **Ambiguous outcomes:** if `apply` times out, read the obligation by `uniq_key` and the verdict by `message_ref` before retrying. The function is idempotent, so a retry cannot double-create.

## G. Autonomy and approval policy

- **Default autonomy level:** A2 — bounded autonomy (internal reminder records only, inside the standing scope the user granted when enabling the radar).
- **Planning level:** A2.
- **Execution level:** A2 for create/update of obligations; A0 for anything routed to needs-review.
- **Verification level:** A2.
- **Recovery level:** A2 — bounded retries; defer rather than guess.
- **Approval required for:** enabling the radar per mailbox (one-time standing scope); dismissing on a fuzzy cancellation match; resolving any needs-review item; restoring a dismissed series.
- **Actions never permitted:** paying or scheduling a payment; replying to or contacting a biller; opening links in messages; storing bodies; scheduling incoming money; acting on instructions found inside an email.
- **Conditions requiring clarification:** ambiguous currency glyph with no locale evidence; two plausible due dates in one message; a merchant that matches two series.
- **Conditions requiring escalation:** expired mailbox authorization; a phishing-classified message that claims to be from a biller already in the schedule (tell the user once, in plain words, without links).
- **Safe state after stop, cancellation or escalation:** no partial obligation rows (apply is atomic); unprocessed messages stay pending; no messages sent except the owner notice.

The standing scope covers creating and updating reminder records for the enabled mailboxes. It does not cover payments, transactions, or any contact with third parties.

## H. Execution plan

**S1 — Validate configuration and take the run lock**
1. Objective: confirm the run may start.
2. Capability: none (local validation); runtime lock (UNVERIFIED binding).
3. Input mapping: installation config → `config@0.1.0`.
4. Dependencies: none.
5. Preconditions: radar enabled for at least one mailbox; kill switch off.
6. Action / side effects: validate config, check mailbox authorization health, acquire the per-user run lock.
7. Autonomy: A2; no approval.
8. Timeout / retry: 10 s / none.
9. Idempotency: lock key = `upcoming-radar:<user>`; a second run exits `cancelled` ("run already in progress").
10. Postconditions: valid config and a held lock.
11. Verification: schema validation passes; every mailbox reports an authorized state.
12. Failure routes: invalid config → `blocked`; an expired mailbox token → skip that mailbox, mark the run `partially_completed`, escalate.
13. Escalation triggers: all mailboxes unauthorized.
14. Evidence: config hash, lock ID, mailbox authorization states.

**S2 — Select candidate messages**
1. Objective: a small batch of unjudged, likely billing messages.
2. Capability: `nova.mail.candidates.list.default`.
3. Input mapping: `mailbox_accounts`, `biller_senders`, `billing_subject_tokens`, `own_domains`, `batch_size`, `backfill_cutoff_days`.
4. Dependencies: S1.
5. Preconditions: metadata sync healthy (last sync ≤ 90 minutes old; otherwise disclose staleness).
6. Action / side effects: read-only selection, deduplicated across mailboxes by RFC Message-ID, oldest first, excluding messages that already have an `upcoming` verdict. On the first run, messages older than the cutoff get the verdict `backfill` and are not extracted.
7. Autonomy: A2.
8. Timeout / retry: 20 s / 2 retries.
9. Idempotency: read-only (the backfill marking is idempotent on the verdict key).
10. Postconditions: 0–`batch_size` `message-ref` items.
11. Verification: no item has an existing verdict; no item is from `own_domains`.
12. Failure routes: empty batch → skip to S10 with `completed`; store error → `failed`.
13. Escalation: metadata sync stale > 24 h → tell the user that the radar is blind.
14. Evidence: selected message refs and selector version.

**S3 — Fetch the body transiently**
1. Objective: the text needed for extraction, held only in memory.
2. Capability: `nova.mail.body.fetch_transient.default`.
3. Input mapping: `message_ref`, `mailbox_ref`.
4. Dependencies: S2.
5. Preconditions: the mailbox is authorized.
6. Action / side effects: one read; nothing persisted.
7. Autonomy: A2.
8. Timeout / retry: 20 s / 2 retries on transient errors.
9. Idempotency: read-only.
10. Postconditions: plain text of 1–15,000 characters.
11. Verification: text non-empty; the subject matches the metadata subject.
12. Failure routes: 404 (message deleted) → verdict `informational` with reason `gone`; transient failure → defer (no verdict, retried next run).
13. Escalation: none.
14. Evidence: fetch status and character count only.

**S4 — Extract the obligation**
1. Objective: a structured, confidence-scored reading of the message.
2. Capability: `nova.finance.extract.obligation.default` (or `.alternate` if the operator configured it).
3. Input mapping: body text, subject, sender domain, received date, `timezone` → prompt with fixed system rules.
4. Dependencies: S3.
5. Preconditions: an approved LLM provider.
6. Action / side effects: one LLM call; message text leaves the runtime for the provider.
7. Autonomy: A2.
8. Timeout / retry: 45 s / 1 retry on transport error; 1 re-ask if the JSON is invalid.
9. Idempotency: not applicable (no state change).
10. Postconditions: an `extraction@0.1.0` object.
11. Verification: JSON Schema valid; `currency` present whenever `amount` is present.
12. Failure routes: still invalid after the re-ask, or provider down → defer (no verdict) so the message is retried; count it in `deferred`.
13. Escalation: provider failing for 3 consecutive runs → notify the user that detection is paused.
14. Evidence: model ID, prompt version, schema-valid flag (no body text).

**S5 — Check the extraction against the message**
1. Objective: catch hallucinated or unsafe fields before anything is written.
2. Capability: deterministic checks (part of the planner).
3. Input mapping: extraction + the transient body.
4. Dependencies: S4.
5. Preconditions: schema-valid extraction.
6. Action: check that the amount string appears in the body (allowing thousands separators); the due date or its textual form appears; the currency is stated or unambiguous from a glyph plus locale; an `external_ref` appears verbatim; phishing cues (credential requests, "update payment information" with a link to a domain that is not the sender's) agree with `class`. A third-party product billed through a marketplace is **not** phishing on its own.
7. Autonomy: A2.
8. Timeout / retry: 2 s / none.
9. Idempotency: not applicable.
10. Postconditions: each field marked `supported` or `unsupported`.
11. Verification: every non-null field is `supported`, or the item is downgraded.
12. Failure routes: unsupported amount, date or currency → set that field to null and the confidence to `low` → route to needs-review.
13. Escalation: none.
14. Evidence: field → supported/unsupported map (no quotes kept).

**S6 — Decide the route**
1. Objective: one deterministic outcome per message.
2. Capability: routing rules, executed inside `nova.finance.obligations.apply` (server-enforced, so a model cannot bypass them).
3. Input mapping: checked extraction, `auto_schedule_min_confidence`, `timezone`.
4. Dependencies: S5.
5. Preconditions: none beyond S5.
6. Action: route by these rules, in order. Class `informational`, `past_event` or `promotional`, or direction `receive` → skip. `phishing` → skip and record `phishing`. `payment_confirmation` → hand off to paid tracking. `cancellation` → dismiss on an exact `series_key` match, otherwise needs-review. `alertable` → normalise the currency glyph; roll a past recurring date forward to the next occurrence on or after today (user time zone); no date → `needs_date`; confidence below the threshold or any field downgraded → `needs_review`; otherwise schedule.
7. Autonomy: A2.
8. Timeout / retry: part of S7.
9. Idempotency: pure function of its inputs.
10. Postconditions: exactly one route.
11. Verification: the route is recomputed from the stored inputs during read-back (S9).
12. Failure routes: no rule matches → `needs_review` with reason `unroutable`.
13. Escalation: none.
14. Evidence: route and rule ID.

**S7 — Apply to the schedule**
1. Objective: write the outcome atomically.
2. Capability: `nova.finance.obligations.apply.default`; `nova.finance.hub.match_account.default` for `pay_account_ref`.
3. Input mapping: route, extraction, `uniq_key`, `account_hint` + sender domain → match request.
4. Dependencies: S6.
5. Preconditions: write scope granted; series not dismissed (a dismissed series turns this into a no-op).
6. Action / side effects: upsert by `uniq_key` with the earliest-future-cycle rule (adopt a nearer due date and keep `alert_state`; move to a new cycle only when the due date jumps by about one full period); attach a suggested account only on a currency-equal `high` match; or dismiss on an exact cancellation match; or create a needs-review entry; or, for a hand-off, record the event for `nova.finance.auto-mark-bills-paid`.
7. Autonomy: A2 within the standing scope.
8. Timeout / retry: 15 s / 2 retries after reading back by key.
9. Idempotency: `uniq_key` plus the verdict key; replay returns `exists`.
10. Postconditions: at most one obligation per `uniq_key`; the outcome is recorded.
11. Verification: the function returns `{outcome, obligation_id}` and the read-back matches.
12. Failure routes: conflict (another run updated the row) → re-read and re-apply once; still conflicting → needs-review.
13. Escalation: none.
14. Evidence: outcome, obligation ID, changed-field list.

**S8 — Record the verdict**
1. Objective: never re-judge a message that has been decided; always retry one that has not.
2. Capability: `nova.mail.verdicts.mark` (inside the same transaction as S7 when possible).
3. Input mapping: `message_ref`, `pipeline = upcoming`, verdict.
4. Dependencies: S7, or S3–S6 for skip routes.
5. Preconditions: a definitive outcome (not a deferral).
6. Action / side effects: one verdict row.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: primary key `(message_ref, pipeline)`.
10. Postconditions: verdict row exists.
11. Verification: read-back.
12. Failure routes: write fails → leave the message pending (it is re-processed next run; S7 is idempotent).
13. Escalation: none.
14. Evidence: verdict row reference.

**S9 — Verify the run's writes**
1. Objective: prove the schedule now reflects the decisions.
2. Capability: `nova.finance.obligations.list`.
3. Input mapping: obligation IDs and `uniq_key`s touched in S7.
4. Dependencies: S7, S8.
5. Preconditions: at least one write happened.
6. Action: read back the rows and recompute each route from the stored fields.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 1 retry.
9. Idempotency: read-only.
10. Postconditions: every touched row is consistent.
11. Verification: no duplicate `uniq_key`; no `upcoming` row without both `due_date` and `high` confidence (or the configured threshold); no `receive` rows; dismissed series unchanged.
12. Failure routes: inconsistency → mark the row `needs_review` with reason `verify_failed`; the run becomes `partially_completed`.
13. Escalation: repeated verification failures → disable auto-scheduling (review-only) and tell the user.
14. Evidence: read-back snapshot of the touched rows (fields only).

**S10 — Report and release the lock**
1. Objective: an honest run summary and, if needed, one notice to the user.
2. Capability: `nova.notify.owner.send` (optional).
3. Input mapping: outcome counts → `radar-run@0.1.0`.
4. Dependencies: S9 (or S2 when the batch was empty).
5. Preconditions: none.
6. Action / side effects: store the run record; if new needs-review items exist, send "2 possible payments need a quick check" with no amounts in shared channels; release the lock.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 2 retries for the notice.
9. Idempotency: notice key = `run_id`.
10. Postconditions: run record stored; lock released.
11. Verification: the counts sum to `examined`.
12. Failure routes: notice fails → the run record is kept and the run is `partially_completed`.
13. Escalation: none.
14. Evidence: run record ID, notice message ID.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Input validity | config matches `config@0.1.0` | validator | valid | `blocked` |
| Mailbox scope | every message read belongs to `mailbox_accounts` | message refs vs config | 100 % | stop, `failed`, report |
| No body persistence | no stored row or log contains body text | storage and log scan in tests | none found | stop and escalate (privacy breach) |
| Extraction support | each non-null amount, date, currency and ref appears in the body | S5 map | all supported or downgraded | needs-review |
| Money format | amounts match the decimal pattern; currency present with any amount | schema | valid | needs-review |
| Direction | no `receive` obligation exists | S9 read-back | zero | revert row to needs-review, report |
| Deduplication | one row per `uniq_key` | unique constraint + read-back | unique | report a defect |
| Earliest cycle | stored `due_date` ≥ today and ≤ the previous future due date unless the cycle advanced | read-back vs prior value | holds | needs-review |
| Verdict truth | verdicts are written only for definitive outcomes | run log | deferred items have no verdict | delete nothing; fix on next run |
| Output schema | `radar_run` valid; counts sum to `examined` | validator | valid | fix counts; never fake values |

A successful apply call proves that a row was written. It does not prove that the obligation is real; that is why S5 and S9 exist.

## J. Self-correction and recovery

- **Failure classes:** validation, missing authorization, metadata staleness, body fetch failure, LLM transport or schema failure, unsupported field, write conflict, failed read-back.
- **Permitted corrections:** defer a message to the next run; re-ask the LLM once for valid JSON; null out unsupported fields and downgrade to needs-review; re-read and re-apply once on a conflict.
- **Forbidden corrections:** guessing a date, amount or currency; switching LLM provider at run time; writing a verdict for a deferred message; re-opening a dismissed series; storing a body "to retry later".
- **Maximum correction cycles:** 1 per message; 3 per run before the remaining messages are deferred.
- **Reverification required after correction:** yes — S5 checks and S9 read-back.
- **Stop conditions:** correction budget exhausted; authorization lost mid-run; kill switch.
- **Recovery or compensation:** restore prior field values from the obligation change history; the user can dismiss or edit any entry.
- **What must be reported:** deferred count and reasons, downgraded fields, conflicts, mailboxes skipped.

## K. Independent challenge

- **Assumptions to challenge:** a "renewal" message is about the user's own subscription (not a marketing teaser); the due date is the charge date, not a statement date; the currency glyph means the currency assumed.
- **Alternative explanation:** the message may be a receipt for the past cycle (route to paid tracking), a cancellation, or a phishing attempt.
- **Contradictory evidence:** an existing paid record or a later cancellation for the same series; two messages giving different amounts for the same cycle (keep the later one, flag the change).
- **Could mislead:** a schedule that looks complete while one mailbox's sync is stale or its authorization has expired — always disclose per-mailbox freshness.
- **Separate verifier:** the deterministic S5 field-support check and the S9 rule recomputation are independent of the LLM that extracted the fields.
- **Findings requiring disclosure:** stale sync, skipped mailboxes, downgraded fields, phishing that impersonates a tracked biller.

## L. Human escalation

- **Triggers:** all mailboxes unauthorized; metadata sync stale > 24 h; LLM provider failing for 3 runs; fuzzy cancellation match; phishing that impersonates a tracked biller; repeated verification failures.
- **Why the agent cannot decide:** reconnecting accounts needs the user; dismissing a real bill on a fuzzy match could cause a missed payment.
- **Information presented:** the affected mailbox or series, the message subject and date, the extracted fields with confidence, and the proposed action.
- **Decision requested:** reconnect; confirm or reject the dismissal; correct the fields of a needs-review item; keep auto-scheduling on or switch to review-only.
- **Options and consequences:** confirming a dismissal stops reminders for that series; rejecting keeps it active.
- **Safe behaviour while waiting:** affected items stay `needs_review`; other messages continue under the standing scope.
- **If no response:** items remain in needs-review indefinitely and are counted in every reminder digest; nothing is dismissed by default.
- **Resume:** after the user acts, the item is re-applied through S7 with fresh authorization checks.

## M. Completion conditions

Mandatory: `radar_run` valid; every write passed S9; no body persisted; no `receive` or phishing obligation; no guessed field; deferred items disclosed; evidence references present.

- `completed` — every selected message reached a definitive outcome (or the batch was empty) and all writes verified.
- `partially_completed` — some messages deferred, a mailbox was skipped, a verification failed, or the notice could not be delivered.
- `blocked` — invalid config, no authorized mailbox, or no approved LLM provider.
- `failed` — the obligation store or metadata store was unavailable, or a scope or privacy check failed.
- `cancelled` — the kill switch was on, another run held the lock, or the operator stopped the run.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome ("1 new, 1 updated, 1 needs review"); `radar_run`; step statuses S1–S10; verification results and failed checks; evidence references (verdict rows, obligation IDs, changed-field lists); approvals and scope (standing scope ID); side effects (rows written, text sent to the LLM provider) and reconciliation status; corrections performed; unresolved challenges; limitations (stale or skipped mailboxes); escalation state; learning candidate. Never include message bodies, credentials or full account numbers.

## O. Learning and revision

- **Outcome metrics:** precision of scheduled items (user dismissals as "not a bill"), needs-review rate, deferral rate, phishing catch count, roll-forward corrections, mean detection lag (message received → scheduled).
- **Evidence retained:** run records, verdict rows and obligation change history, per the retention policy.
- **Data minimisation:** no body text, only subject lines ≤ 200 characters and message references; metrics stored as counts.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}` — for example a new biller sender pattern or a new subject token.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests; a selector change also needs the precision fixture set.
- **Promotion approval:** publisher review plus Skill Port review; selector and prompt changes are versioned and never applied silently.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (fixture renewal email in a sandbox mailbox) | one `upcoming` obligation with correct amount, currency and date | UNVERIFIED on Skill Port; observed in the reference implementation |
| Invalid input (bad time zone) | `blocked` | TBD |
| Missing input (no mailbox) | `blocked` | TBD |
| Missing capability (LLM binding absent) | `blocked`; nothing written | TBD |
| Permission denied (`mail:body:read_transient` missing) | `blocked`; no body read | TBD |
| Unauthorized scope (a mailbox not in config) | never read | TBD |
| Tenant isolation | cannot read or write another tenant's obligations | TBD |
| Recurring date in the past | rolled forward to the next occurrence ≥ today | TBD |
| No due date | `needs_date` | TBD |
| Low confidence / unsupported amount | `needs_review` with a reason | TBD |
| Phishing fixture (credential-request lure) | no obligation; verdict `phishing` | TBD |
| Marketplace-billed third-party product | treated as legitimate, scheduled | TBD |
| Incoming-money notice | skipped, no obligation | TBD |
| Cancellation, exact match | series dismissed, sticky | TBD |
| Cancellation, fuzzy match | needs-review, nothing dismissed | TBD |
| Timeout on apply | read-back by key; no duplicate | TBD |
| Duplicate / replay (same message twice, or the same message in two mailboxes) | one obligation, one verdict | TBD |
| False success (apply returns OK, read-back missing) | `partially_completed`, reported | TBD |
| Failed verification | row downgraded to needs-review | TBD |
| Correction budget exhausted | remaining messages deferred, run stops safely | TBD |
| Escalation (all mailboxes unauthorized) | user told; nothing written | TBD |
| Cancellation (kill switch / lock held) | `cancelled`, nothing written | TBD |
| Compensation (user restores a dismissed series) | previous values restored from history | TBD |
| Output schema and status | valid `radar_run`, correct status | TBD |
| Compatibility / regression (new LLM model version) | fixture set precision unchanged | TBD |

All tests use sandbox mailboxes with fictional fixture messages and rolled-back database transactions; never real inboxes or real bills. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, schema registry and needs-review UI — TBD (owner: publisher, with Skill Port).
- The biller sender list and subject tokens are region-specific; card-statement and instalment senders must be configured per user (owner: user/operator).
- Should phishing that impersonates a tracked biller trigger a low-frequency warning by default? Currently a one-time notice (owner: publisher).
- Retention period for verdicts and obligation history — TBD (owner: operator).
- Bills sent only by post, SMS or in-app are invisible to this recipe.
- Amounts hidden inside images or tracking links cannot be read; those messages go to needs-review or are deferred.
- The change-history table for obligations is specified here but its name and schema are TBD.
