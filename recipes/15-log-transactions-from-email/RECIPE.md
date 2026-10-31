# Log transactions from email

## A. Identity and ownership

- **Recipe ID:** `nova.finance.log-transactions-from-email` (proposed until assigned by Skill Port)
- **Name:** Log transactions from email
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** finance
- **Tags:** transactions, bank-alerts, card-spend, email-extraction, finance-hub, review-queue, deduplication · Vedaxi task type: none (no supported task type covers this today)
- **Summary:** On a schedule, the agent reads transaction emails (bank debit/credit alerts, card spends, settlements, refunds) and extracts each one with an LLM. It logs the transaction to the finance hub only when the account match and the model's confidence are both high, inside a standing scope the user granted. Everything else waits in a review queue for the user's approval. Per-order payment-gateway notices are skipped, and an existing manual entry within ±3 days is detected, so nothing is counted twice.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** Mailbox metadata sync and on-demand body reads; a structured-output LLM; the finance hub (records, accounts with match hints, review queue); a sync adapter that mirrors records to an external personal-finance app over REST. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — extraction, the high + high auto-log gate, the review queue with approval, gateway-notice skipping, the ±3-day manual-duplicate reconcile and the claim → push → mark sync have run end-to-end in Nova's reference implementation (catalogue status: Built). Not every control described here has evidence yet (Section P lists the tests still needed). Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.mail.candidates.list` | Operation (read) | Unjudged messages matching the transaction selector | medium | UNVERIFIED |
| `nova.mail.body.fetch_transient` | Operation (read) | Message text held only in memory | medium | UNVERIFIED |
| `nova.mail.verdicts.mark` | Operation (create) | Record the definitive verdict (`pipeline = txn`) | low | UNVERIFIED |
| `nova.finance.extract.transaction` | Operation (read, LLM) | Extract one transaction with confidence and skip reason | medium | UNVERIFIED |
| `nova.finance.hub.match_account` | Operation (read) | Match account hints and sender to one account: `high/medium/low/none` | medium | UNVERIFIED |
| `nova.finance.hub.find_duplicate` | Operation (read) | Find an existing manual entry (±3 days) or a same-reference record from any source | low | UNVERIFIED |
| `nova.finance.hub.route_transaction` | Operation (create) | Atomic router: stage a record, add a review item, or reconcile — plus the verdict | high | UNVERIFIED |
| `nova.finance.hub_sync.push_batch` | Operation (external side effect) | Claim staged records and create them in the external personal-finance app | high | UNVERIFIED |
| `nova.finance.hub_sync.verify_remote` | Operation (read) | Look up a record in the external app before any re-push | low | UNVERIFIED |
| `nova.finance.review.approve` | Operation (create, user-approved) | Log a reviewed item to the account the user picks | medium | UNVERIFIED |
| `nova.finance.review.reject` | Operation (update, user) | Close a review item without logging | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Tell the user that review items are waiting | low | UNVERIFIED |

**Capabilities (object type Capability, all UNVERIFIED):** `nova.mail.candidates` (message selection, medium), `nova.mail.body` (transient body, medium), `nova.mail.verdicts` (judged-message memory, low), `nova.finance.extract` (LLM extraction, medium), `nova.finance.hub` (accounts, records, routing, high), `nova.finance.hub_sync` (mirror to the external personal-finance app, high), `nova.finance.review` (approval queue, medium), `nova.notify.owner` (owner channel, low).

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.mail.candidates.list.txn` — sender on the configured institution list, or a transaction token in the subject (debited, credited, spent, payment of, settlement, refund). Excludes the user's own domains, newsletters and one-time-code messages. Deduplicated by RFC Message-ID; batch default 8.
- `nova.finance.extract.transaction.default` / `.alternate` — as in the radar recipe: temperature 0, strict JSON. The alternate is chosen by operator configuration only and is never a runtime fallback.
- `nova.finance.hub.match_account.default` — institution keyword and last-4 hints per account; currency-gated. An institution with more than one account must match on last-4 to reach `high`.
- `nova.finance.hub.route_transaction.default` — one security-definer function holds the money-routing decision in the audited database layer, not in workflow branches.
- `nova.finance.hub_sync.push_batch.default` — at most 20 records per call. Results are matched by input index. Only `pending` rows are claimed (`pending → pushing`, skip-locked). Where the external app's API accepts no client idempotency key, duplicates are prevented by this claim protocol plus `verify_remote`.
- `nova.finance.review.approve.default` — the user selects the account; the item is logged with `source_type = review`.

**Bindings:** `binding://nova/mail/candidates`, `binding://nova/mail/body-transient`, `binding://nova/mail/verdicts`, `binding://nova/finance/extract-transaction`, `binding://nova/finance/match-account`, `binding://nova/finance/find-duplicate`, `binding://nova/finance/route-transaction`, `binding://nova/finance/hub-push`, `binding://nova/finance/hub-verify`, `binding://nova/finance/review-approve`, `binding://nova/finance/review-reject`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: a 30-minute automation-hub workflow calls a webhook body fetcher with execution saving disabled, then the LLM, then a database router function. A 5-minute pusher syncs records to the external app over REST.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Message metadata / body | `message-ref@0.1.0` / transient | connected mailboxes | read (body never stored) |
| Hub record | `schema://nova/log-transactions-from-email/hub-record@0.1.0` | the requesting user's tenant | create; update (`sync_status`, `remote_id`, `error` only) |
| Review item | `schema://nova/log-transactions-from-email/review-item@0.1.0` | the requesting user's tenant | create; update (`status`, `resolved_account_ref`) |
| Account | `account-ref@0.1.0` | the requesting user | read |
| External app record | provider-defined; mirrored | the user's external app account | create (via sync only) |

**Authorization requirements (declared, never a grant):** `mail:metadata:read`, `mail:body:read_transient`, `mail:verdicts:write`, `llm:extract`, `finance:hub:accounts:read`, `finance:hub:records:read`, `finance:hub:records:write`, `finance:review:write`, `external_finance_app:records:create`, `external_finance_app:records:read`, `notify:owner`. Secrets: `secret_ref://mail-oauth`, `secret_ref://llm`, `secret_ref://finance-app-api`, `secret_ref://finance-db-role`.

**Policy / trust:** risk class `high` (writes financial records that change displayed balances). Auto-logging needs a standing approval record naming accounts and a per-transaction cap. Email text is untrusted.

## C. Intent and applicability

- **User problem:** Typing every card spend and bank credit into a finance app by hand is tedious, and messages arrive in different inboxes.
- **Desired outcome:** Clear-cut transactions appear in the finance hub automatically and exactly once. Unclear ones wait for a one-tap approval.
- **Trigger or entry conditions:** Scheduled run (default every 30 minutes) after the user has enabled the recipe and granted the auto-log scope; or "check my mail for transactions now".
- **Applicable situations:** Bank and card debit/credit alerts, e-wallet payments, payment-gateway **settlements** credited to a bank, refunds to a tracked account.
- **Out-of-scope situations:** Per-order "payment received" notices from a payment gateway (money not yet in a tracked account); statements; refunds to gift cards or store credit; bills not yet paid (`nova.finance.upcoming-payments-radar`); moving money; categorising historical exports.
- **Required inputs:** `mailbox_accounts`, `timezone`, `autolog_scope` (see Section D).
- **Prerequisites:** Accounts set up in the hub with match hints (last-4 for multi-account institutions); external app connection authorized; operator-approved LLM provider.
- **Required capabilities:** listed in Section B (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port; every binding is a dependency to resolve before activation.
- **Assumptions:** Verified in the reference implementation: the high + high gate, gateway-notice skip, gift-card skip, manual-duplicate reconcile, and backlog routed to review instead of auto-logged. Not every control described here has evidence yet (Section P lists the tests still needed).
- **Ambiguity policy:** Ask, through the review queue. Never guess an account, amount, currency or direction.
- **Vedaxi task type:** none.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `mailbox_accounts` | array of refs, 1–10 | yes | installation; personal | — | `blocked`. |
| `timezone` | IANA zone | yes | profile | — | `blocked`. |
| `autolog_scope` | `{approval_id, accounts[], max_amount:{CUR:"decimal"}, enabled_at}` | yes (may list zero accounts = review-only) | user approval record | review-only | Missing or expired approval → review-only for every item. |
| `batch_size` | integer 1–20 | no | operator | 8 | Clamp. |
| `institution_senders` | array of sender patterns | no | user | empty | Subject tokens only. |
| `dup_window_days` | integer, fixed 3 | — | recipe constant | 3 | Not configurable in 0.1.0. |

```json
{"$id":"schema://nova/log-transactions-from-email/extraction@0.1.0","type":"object","required":["is_transaction","confidence"],
 "properties":{"is_transaction":{"type":"boolean"},"direction":{"enum":["expense","income"]},
 "amount":{"type":["string","null"],"pattern":"^[0-9]{1,13}(\\.[0-9]{1,4})?$"},"currency":{"type":["string","null"],"pattern":"^[A-Z]{3}$"},
 "txn_date":{"type":["string","null"],"format":"date"},"counterparty":{"type":["string","null"],"maxLength":120},
 "account_hint":{"type":["string","null"],"maxLength":64},"reference":{"type":["string","null"],"maxLength":64},
 "credited_to":{"enum":["tracked_account","gift_card_or_store_credit","payment_gateway_balance","unknown",null]},
 "category":{"type":["string","null"],"maxLength":60},"confidence":{"enum":["high","medium","low"]},
 "skip_reason":{"enum":["not_transaction","gateway_order_notice","statement","one_time_code","promotional","onboarding","store_credit","amount_unreadable",null]}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `txn_run` | `schema://nova/log-transactions-from-email/txn-run@0.1.0` | `run_id`, `status`, `examined`, `outcomes` | run record; owner channel for review counts | financial; owner only | counts reconcile with verdicts, records and review rows |
| Hub records / review items | schemas below | — | finance hub | financial | read-back; remote ID after sync |

```json
{"$id":"schema://nova/log-transactions-from-email/hub-record@0.1.0","type":"object",
 "required":["record_id","account_ref","direction","amount","currency","txn_date","source_type","source_ref","sync_status"],
 "properties":{"record_id":{"type":"string"},"account_ref":{"type":"string"},"direction":{"enum":["expense","income"]},
 "amount":{"type":"string","pattern":"^[0-9]{1,13}(\\.[0-9]{1,4})?$"},"currency":{"type":"string","pattern":"^[A-Z]{3}$"},
 "txn_date":{"type":"string","format":"date"},"counterparty":{"type":["string","null"]},"category":{"type":["string","null"]},
 "source_type":{"enum":["mail","sms","review","manual"]},"source_ref":{"type":"string"},
 "sync_status":{"enum":["pending","pushing","pushed","failed"]},"remote_id":{"type":["string","null"]},"claimed_at":{"type":["string","null"],"format":"date-time"}}}
```

```json
{"$id":"schema://nova/log-transactions-from-email/review-item@0.1.0","type":"object",
 "required":["review_id","source_ref","direction","amount","currency","reason","status"],
 "properties":{"review_id":{"type":"string"},"source_ref":{"type":"string"},"direction":{"enum":["expense","income"]},
 "amount":{"type":"string","pattern":"^[0-9]{1,13}(\\.[0-9]{1,4})?$"},"currency":{"type":"string","pattern":"^[A-Z]{3}$"},
 "counterparty":{"type":["string","null"]},"txn_date":{"type":["string","null"],"format":"date"},
 "reason":{"enum":["account_match_not_high","confidence_not_high","account_not_opted_in","over_cap","backlog","possible_duplicate"]},
 "suggested_account_ref":{"type":["string","null"]},"subject":{"type":"string","maxLength":200},
 "status":{"enum":["pending","approved","rejected"]},"resolved_account_ref":{"type":["string","null"]},"record_id":{"type":["string","null"]}}}
```

`source_ref` = the bank reference when present (`ref:<normalised>`), else `mail:<message_ref>`; it is unique per `(source_type, source_ref)`. Supporting schemas (one per line; several are reused by `nova.finance.log-transactions-from-sms`):

```json
{"$id":"schema://nova/log-transactions-from-email/config@0.1.0","type":"object","required":["mailbox_accounts","timezone","autolog_scope"],"properties":{"mailbox_accounts":{"type":"array","minItems":1,"maxItems":10,"items":{"type":"string"}},"timezone":{"type":"string"},"autolog_scope":{"type":"object","required":["approval_id","accounts","max_amount","enabled_at"],"properties":{"approval_id":{"type":["string","null"]},"accounts":{"type":"array","items":{"type":"string"}},"max_amount":{"type":"object","additionalProperties":{"type":"string","pattern":"^[0-9]{1,13}(\\.[0-9]{1,4})?$"}},"enabled_at":{"type":"string","format":"date-time"}}},"batch_size":{"type":"integer","minimum":1,"maximum":20},"institution_senders":{"type":"array","items":{"type":"string"}}}}
{"$id":"schema://nova/log-transactions-from-email/message-ref@0.1.0","type":"object","required":["message_ref","mailbox_ref","received_at"],"properties":{"message_ref":{"type":"string"},"mailbox_ref":{"type":"string"},"received_at":{"type":"string","format":"date-time"},"from_domain":{"type":"string"},"subject":{"type":"string","maxLength":200}}}
{"$id":"schema://nova/log-transactions-from-email/body-text@0.1.0","type":"object","required":["message_ref","text"],"description":"transient, never persisted","properties":{"message_ref":{"type":"string"},"subject":{"type":"string","maxLength":200},"text":{"type":"string","minLength":1,"maxLength":15000}}}
{"$id":"schema://nova/log-transactions-from-email/verdict@0.1.0","type":"object","required":["message_ref","pipeline","verdict"],"properties":{"message_ref":{"type":"string"},"pipeline":{"const":"txn"},"verdict":{"enum":["auto_logged","review","reconciled","not_transaction","gateway_order_notice","statement","one_time_code","promotional","onboarding","store_credit","amount_unreadable"]},"judged_at":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/log-transactions-from-email/account-hint@0.1.0","type":"object","properties":{"account_hint":{"type":["string","null"],"maxLength":64},"sender":{"type":["string","null"],"maxLength":120},"currency":{"type":["string","null"],"pattern":"^[A-Z]{3}$"}}}
{"$id":"schema://nova/log-transactions-from-email/account-ref@0.1.0","type":"object","required":["match"],"properties":{"account_ref":{"type":["string","null"]},"display_name":{"type":["string","null"]},"currency":{"type":["string","null"],"pattern":"^[A-Z]{3}$"},"last4":{"type":["string","null"],"pattern":"^[0-9]{4}$"},"match":{"enum":["high","medium","low","none"]},"hint_type":{"enum":["last4","institution","sender_id",null]}}}
{"$id":"schema://nova/log-transactions-from-email/dup-query@0.1.0","type":"object","required":["direction","amount","currency","txn_date"],"properties":{"account_ref":{"type":["string","null"]},"direction":{"enum":["expense","income"]},"amount":{"type":"string","pattern":"^[0-9]{1,13}(\\.[0-9]{1,4})?$"},"currency":{"type":"string","pattern":"^[A-Z]{3}$"},"txn_date":{"type":"string","format":"date"},"reference":{"type":["string","null"]},"window_days":{"const":3}}}
{"$id":"schema://nova/log-transactions-from-email/dup-result@0.1.0","type":"object","required":["result"],"properties":{"result":{"enum":["none","exact","multiple"]},"matched_record_ids":{"type":"array","items":{"type":"string"}},"match_basis":{"enum":["manual_window","same_reference",null]}}}
{"$id":"schema://nova/log-transactions-from-email/route-input@0.1.0","type":"object","required":["source_type","source_ref","extraction","account_match","duplicate","approval_id"],"properties":{"source_type":{"enum":["mail","sms"]},"source_ref":{"type":"string"},"message_ref":{"type":["string","null"]},"received_at":{"type":"string","format":"date-time"},"extraction":{"$ref":"schema://nova/log-transactions-from-email/extraction@0.1.0"},"account_match":{"$ref":"schema://nova/log-transactions-from-email/account-ref@0.1.0"},"duplicate":{"$ref":"schema://nova/log-transactions-from-email/dup-result@0.1.0"},"approval_id":{"type":["string","null"]}}}
{"$id":"schema://nova/log-transactions-from-email/route-result@0.1.0","type":"object","required":["outcome"],"properties":{"outcome":{"enum":["auto_logged","review","reconciled","skipped","exists"]},"record_id":{"type":["string","null"]},"review_id":{"type":["string","null"]},"reason":{"type":["string","null"]},"rule_id":{"type":"string"}}}
{"$id":"schema://nova/log-transactions-from-email/push-result@0.1.0","type":"object","required":["results"],"properties":{"results":{"type":"array","items":{"type":"object","required":["record_id","input_index","outcome"],"properties":{"record_id":{"type":"string"},"input_index":{"type":"integer","minimum":0},"outcome":{"enum":["pushed","failed","pending","pushing"]},"remote_id":{"type":["string","null"]},"error":{"type":["string","null"]},"retry_after_s":{"type":["integer","null"]}}}}}}
{"$id":"schema://nova/log-transactions-from-email/review-decision@0.1.0","type":"object","required":["review_id","decision"],"properties":{"review_id":{"type":"string"},"decision":{"enum":["approve","reject"]},"account_ref":{"type":["string","null"],"description":"required when approving"}}}
{"$id":"schema://nova/log-transactions-from-email/txn-run@0.1.0","type":"object","required":["run_id","status","examined","outcomes"],"properties":{"run_id":{"type":"string"},"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},"examined":{"type":"integer","minimum":0},"outcomes":{"type":"object","properties":{"auto_logged":{"type":"integer"},"review":{"type":"integer"},"reconciled":{"type":"integer"},"skipped":{"type":"integer"},"amount_unreadable":{"type":"integer"},"deferred":{"type":"integer"}}},"pushed":{"type":"integer"},"push_failed":{"type":"integer"},"limitations":{"type":"array","items":{"type":"string"}}}}
{"$id":"schema://nova/log-transactions-from-email/delivery@0.1.0","type":"object","required":["delivered"],"properties":{"delivered":{"type":"boolean"},"message_id":{"type":["string","null"]},"error":{"type":["string","null"]}}}
```

**Example (fictional)**

Email: "Example Bank: USD 42.50 spent on card ending 0142 at Example Grocer on 03 Nov 2026. Ref TXN88213."

Output item: `{"record_id":"rec_0101","account_ref":"acct_ex_card_01","direction":"expense","amount":"42.50","currency":"USD","txn_date":"2026-11-03","counterparty":"Example Grocer","category":"groceries","source_type":"mail","source_ref":"ref:TXN88213","sync_status":"pushed","remote_id":"ext_ex_5521","claimed_at":"2026-11-03T14:05:00Z"}`. Run: `{"run_id":"run_0207","status":"completed","examined":3,"outcomes":{"auto_logged":1,"review":1,"reconciled":0,"skipped":1,"amount_unreadable":0,"deferred":0},"pushed":1,"push_failed":0,"limitations":[]}`

## E. Constraints and budgets

- **Hard constraints:** never move money or initiate payments; auto-log only when account match = `high`, model confidence = `high`, the account is in `autolog_scope.accounts`, the amount is ≤ the cap for its currency, and the message arrived after `enabled_at`; everything else goes to review; amounts are decimal strings with an explicit currency (converted to the external API's number format with exact decimal arithmetic); skip gateway per-order notices, statements, one-time codes, promotions, onboarding and store-credit refunds; a manual entry with the same account, direction and exact amount within ±3 days, or a same-reference record from any source, means `reconciled` (no new record); never store bodies; never edit or delete manual records.
- **Soft constraints:** keep counterparty names as written; suggest a category.
- **Resource and tenant scope:** the requesting user's mailboxes, hub records, review items and external app account.
- **Data sources allowed:** message metadata and transient bodies, hub accounts and records.
- **External transfers permitted:** message text to the approved LLM provider; record fields (amount, currency, date, account, counterparty, note) to the user's external finance app.
- **Time limit:** 10 minutes per run.
- **Cost or token budget:** ≤ `batch_size` LLM calls per run; token cap `TBD`; external API calls within its rate limit (`TBD` per provider; the design assumes a few hundred per hour).
- **Maximum steps:** 11 (S1–S11), S3–S9 per message.
- **Maximum retries and correction cycles:** body 2, LLM 1 + 1 re-ask, route 2 (idempotent), push: never re-POST a `pushing` row without `verify_remote`; correction cycles 1 per message.
- **Rate limits and concurrency:** one run per user; one pusher at a time (skip-locked claims).
- **Retention and logging constraints:** records and review items are the user's financial history (retention per operator policy, `TBD`); logs never contain bodies, credentials or full account numbers.
- **Prohibited actions:** guessing an account; logging gift-card credits; logging backlog items automatically; posting to the external app outside the claim protocol.
- **Stop conditions:** auto-log approval expired (switch to review-only); external app unauthorized (stage only, push later); kill switch.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Hub record (`source_type = mail`) | all auto-log conditions hold | Allowed within the standing auto-log approval | read-back by `source_ref` |
| Create | Hub record (`source_type = review`) | user approved the review item and chose the account | Approval Required (per item) | read-back |
| Create | Review item | any other valid transaction | Allowed | row exists |
| Create | Verdict | definitive outcome | Allowed | row exists |
| Read | Mail metadata/body (transient), accounts, records | own tenant | Allowed | — |
| Update | Hub record `sync_status`, `remote_id`, `claimed_at`, `error` | sync protocol only | Allowed | read-back |
| Update | Review item `status`, `resolved_account_ref`, `record_id` | user action | Approval Required | read-back |
| Delete | records, review items, verdicts, external records | — | Not Allowed (a wrong record is reversed by the user in the hub; see compensation) | — |
| External side effect | record created in the external app | `pending` claimed to `pushing` | Allowed (covered by the record's own approval) | remote ID returned |

- **Idempotency:** `(source_type, source_ref)` is unique in the hub; review items are unique on `source_ref`; the sync never re-POSTs a `pushing` row older than 15 minutes without `verify_remote` finding no remote copy.
- **Expected side effects:** hub records, review items and verdicts written; external records created; one optional notice.
- **Compensation:** the user rejects or reverses an entry in the hub; the recipe records an offsetting `reversal` only on the user's explicit request (Approval Required), never on its own.
- **Irreversible:** text sent to the LLM; a record already visible in the external app until the user reverses it.
- **Ambiguous outcomes:** push timeout or 5xx → leave `pushing`; after 15 minutes `verify_remote` decides `pushed` (if found) or back to `pending`.

## G. Autonomy and approval policy

- **Default autonomy level:** A1 — supervised execution. Auto-logging is a narrowly bounded A2 exception inside the standing approval (named accounts, per-transaction cap, high + high).
- **Planning level:** A2.
- **Execution level:** A2 inside `autolog_scope`; A0 for review items.
- **Verification level:** A2.
- **Recovery level:** A2 for reads and idempotent writes; A1 for anything ambiguous.
- **Approval required for:** granting or renewing `autolog_scope`; every review item; reversals.
- **Actions never permitted:** moving money; logging to a guessed account; logging gateway per-order notices or store credit; auto-logging backlog; deleting records.
- **Conditions requiring clarification:** a transaction that could belong to two accounts; a currency that differs from the matched account's currency.
- **Conditions requiring escalation:** the external app rejects records repeatedly; review queue > 25 items for 7 days.
- **Safe state after stop, cancellation or escalation:** staged records stay `pending`; review items stay pending; no partial routes (the router is atomic).

The standing approval covers only the named accounts, up to the cap, for high + high items. It never covers backlog, other accounts or other recipes.

## H. Execution plan

**S1 — Validate configuration and scope; take the lock**
1. Objective: know exactly what may be auto-logged in this run.
2. Capability: none (local validation); approval store read (UNVERIFIED binding).
3. Input mapping: installation config, `autolog_scope`.
4. Dependencies: none.
5. Preconditions: kill switch off.
6. Action / side effects: validate; load the approval record; acquire `txn-email:<user>`.
7. Autonomy: A2.
8. Timeout / retry: 10 s / none.
9. Idempotency: run lock.
10. Postconditions: effective scope (possibly review-only).
11. Verification: approval unexpired and owned by the user; accounts exist.
12. Failure routes: invalid config → `blocked`; expired approval → review-only and disclose.
13. Escalation triggers: approval expired → ask to renew.
14. Evidence: approval ID, effective account list, caps.

**S2 — Select candidate messages**
1. Objective: unjudged transaction messages.
2. Capability: `nova.mail.candidates.list.txn`.
3. Input mapping: `mailbox_accounts`, `institution_senders`, `batch_size`.
4. Dependencies: S1.
5. Preconditions: metadata sync fresh (≤ 90 min, else disclose).
6. Action / side effects: read-only.
7. Autonomy: A2.
8. Timeout / retry: 20 s / 2.
9. Idempotency: read-only.
10. Postconditions: 0–`batch_size` refs.
11. Verification: none already has a `txn` verdict.
12. Failure routes: empty → S10; store error → `failed`.
13. Escalation: sync stale > 24 h → tell the user.
14. Evidence: refs, selector version.

**S3 — Fetch the body transiently**
1. Objective: text for extraction, memory only.
2. Capability: `nova.mail.body.fetch_transient.default`.
3. Input mapping: `message_ref`.
4. Dependencies: S2.
5. Preconditions: mailbox authorized.
6. Action / side effects: read; nothing persisted.
7. Autonomy: A2.
8. Timeout / retry: 20 s / 2.
9. Idempotency: read-only.
10. Postconditions: text ≤ 15,000 characters.
11. Verification: non-empty; the subject matches.
12. Failure routes: gone → verdict `not_transaction`; transient → defer (no verdict).
13. Escalation: none.
14. Evidence: status and length only.

**S4 — Extract the transaction**
1. Objective: structured fields with confidence.
2. Capability: `nova.finance.extract.transaction`.
3. Input mapping: body, subject, sender domain, received date, account names (no numbers beyond last-4).
4. Dependencies: S3.
5. Preconditions: approved provider.
6. Action / side effects: one LLM call.
7. Autonomy: A2.
8. Timeout / retry: 45 s / 1 + 1 re-ask.
9. Idempotency: no state change.
10. Postconditions: `extraction@0.1.0`.
11. Verification: schema valid; currency present whenever amount present.
12. Failure routes: still invalid / provider down → defer (no verdict).
13. Escalation: 3 failing runs → tell the user that capture is paused.
14. Evidence: model ID, prompt version, valid flag.

**S5 — Check fields and apply skip rules**
1. Objective: drop non-transactions and hallucinated fields.
2. Capability: deterministic checks.
3. Input mapping: extraction + body.
4. Dependencies: S4.
5. Preconditions: valid extraction.
6. Action: verify that the amount, last-4 and reference appear in the body; skip if `is_transaction = false` or `skip_reason` is set; skip if `credited_to` is `gift_card_or_store_credit` or `payment_gateway_balance` (per-order gateway notices); amount not readable → `amount_unreadable`.
7. Autonomy: A2.
8. Timeout / retry: 2 s / none.
9. Idempotency: pure.
10. Postconditions: a supported transaction, or a skip verdict.
11. Verification: every non-null money field is supported by the body.
12. Failure routes: unsupported amount → `amount_unreadable`; unsupported account hint → null it (the item will route to review).
13. Escalation: none.
14. Evidence: support map, skip reason.

**S6 — Match the account**
1. Objective: exactly one account, or an honest "not sure".
2. Capability: `nova.finance.hub.match_account.default`.
3. Input mapping: `account_hint` + sender domain + `currency`.
4. Dependencies: S5.
5. Preconditions: accounts configured.
6. Action / side effects: read-only.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2.
9. Idempotency: read-only.
10. Postconditions: `{account_ref|null, match}`.
11. Verification: a `high` match is unique and its currency equals the transaction currency.
12. Failure routes: two candidates → `medium` with the best suggestion; currency mismatch → `low`.
13. Escalation: none.
14. Evidence: match level and the hint type used (last-4 or institution).

**S7 — Check for duplicates**
1. Objective: never count the same money twice.
2. Capability: `nova.finance.hub.find_duplicate`.
3. Input mapping: account, direction, amount, currency, `txn_date`, `source_ref`.
4. Dependencies: S6.
5. Preconditions: amount and date known.
6. Action / side effects: read-only. Look for (a) a manual record with the same account, direction and exact amount within ±3 days, and (b) any record (mail, SMS, review) with the same normalised reference.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2.
9. Idempotency: read-only.
10. Postconditions: `none`, `exact` (one match) or `multiple`.
11. Verification: the matched record belongs to the tenant.
12. Failure routes: `multiple` → review with reason `possible_duplicate`.
13. Escalation: none.
14. Evidence: matched record ID.

**S8 — Decide the route**
1. Objective: one outcome per message.
2. Capability: rules enforced inside `nova.finance.hub.route_transaction` (server-side).
3. Input mapping: S5–S7 results, effective scope, `enabled_at`.
4. Dependencies: S7.
5. Preconditions: none.
6. Action: `exact` duplicate → `reconciled`; message older than `enabled_at` → review `backlog`; all auto-log conditions → `auto_log`; otherwise review with the first failing reason.
7. Autonomy: A2.
8. Timeout / retry: part of S9.
9. Idempotency: pure.
10. Postconditions: route + reason.
11. Verification: recomputed server-side; a mismatch aborts.
12. Failure routes: none.
13. Escalation: none.
14. Evidence: route, reason, rule ID.

**S9 — Write the route atomically**
1. Objective: stage, queue or reconcile — together with the verdict.
2. Capability: `nova.finance.hub.route_transaction.default`.
3. Input mapping: route, fields, account, `source_ref`.
4. Dependencies: S8.
5. Preconditions: write scopes.
6. Action / side effects: insert a `pending` hub record, a review item, or nothing (reconciled), plus the verdict, in one transaction.
7. Autonomy: A2 inside scope.
8. Timeout / retry: 15 s / 2 after read-back by `source_ref`.
9. Idempotency: unique `(source_type, source_ref)`; returns `exists` on replay.
10. Postconditions: exactly one outcome stored.
11. Verification: returned outcome matches the read-back.
12. Failure routes: conflict → re-run S7–S9 once → else review.
13. Escalation: none.
14. Evidence: record or review ID, verdict.

**S10 — Sync staged records to the external app**
1. Objective: mirror new records exactly once.
2. Capability: `nova.finance.hub_sync.push_batch`, `nova.finance.hub_sync.verify_remote`.
3. Input mapping: up to 20 `pending` records → provider payload (signed value from direction + amount, account mapping, date, note with `source_ref`).
4. Dependencies: S9 (also runs on its own 5-minute schedule).
5. Preconditions: external app authorized.
6. Action / side effects: claim `pending → pushing`; POST; per input index: success → `pushed` + `remote_id`; validation error → `failed`; conflict or rate-limit → back to `pending` (honour Retry-After); 5xx or timeout → stay `pushing`. Stale `pushing` (> 15 min) → `verify_remote` → `pushed` or `pending`.
7. Autonomy: A2.
8. Timeout / retry: 30 s per batch; no blind re-POST.
9. Idempotency: claim protocol + `source_ref` marker in the note + `verify_remote`.
10. Postconditions: each claimed record `pushed`, `failed` or safely `pending`.
11. Verification: count of `pushed` = count of successes in the response; remote IDs non-null.
12. Failure routes: unauthorized → leave `pending`, escalate; repeated `failed` → review.
13. Escalation: 3 runs of failures → tell the user.
14. Evidence: remote IDs, response summary.

**S11 — Verify and report**
1. Objective: honest totals and a nudge for review items.
2. Capability: `nova.notify.owner.send` (optional).
3. Input mapping: outcome counts → `txn-run@0.1.0`.
4. Dependencies: S9, S10.
5. Preconditions: none.
6. Action / side effects: read back touched rows; store the run; if new review items exist, send "2 transactions need your OK" (no amounts in shared channels); release the lock.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 2.
9. Idempotency: notice key `run_id`.
10. Postconditions: run stored; lock released.
11. Verification: counts sum to `examined`; no auto-logged record breaks a scope rule.
12. Failure routes: notice fails → `partially_completed`.
13. Escalation: none.
14. Evidence: run ID, notice ID.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Scope | every auto-logged record: high + high, opted-in account, ≤ cap, after `enabled_at` | S11 read-back vs approval | 100 % | move the record to review, report a defect |
| Field support | amount, last-4 and reference appear in the body | S5 map | supported | review / `amount_unreadable` |
| Skip rules | no gateway-order, store-credit or statement item logged | verdicts vs records | none | report defect |
| Duplicates | no two records for one reference; manual duplicates reconciled | S7 + unique constraint | holds | reverse via user; report |
| Money format | decimal strings + currency; exact conversion to the provider payload | validator + payload diff | exact | block the push |
| No body persistence | no body text in rows or logs | storage scan in tests | none | escalate (privacy) |
| Sync truth | `pushed` only with a remote ID | sync log | holds | revert to `pushing`, verify |
| Output schema | `txn_run` valid | validator | valid | fix; never fake counts |

A 200 from the external API proves the batch was accepted. Each record is `pushed` only when its own result entry succeeded.

## J. Self-correction and recovery

- **Failure classes:** validation, expired approval, sync staleness, body/LLM failure, unsupported field, account ambiguity, duplicate ambiguity, write conflict, push failure (validation, rate limit, 5xx, timeout), external authorization loss.
- **Permitted corrections:** defer; re-ask the LLM once; downgrade to review; re-run S7–S9 once on a conflict; honour Retry-After; verify before re-push.
- **Forbidden corrections:** lowering the gate; guessing an account; re-POSTing a `pushing` row blindly; deleting records; auto-logging backlog.
- **Maximum correction cycles:** 1 per message; 3 per run.
- **Reverification required after correction:** yes — S5–S7 and S11 read-back.
- **Stop conditions:** budget exhausted; kill switch; authorization lost.
- **Recovery or compensation:** user-approved reversal in the hub.
- **What must be reported:** deferrals, review reasons, reconciled duplicates, push failures.

## K. Independent challenge

- **Assumptions to challenge:** the alert is the user's own account (not a family member's forwarded alert); a credit is income rather than a refund or an own-account transfer; the currency is the account currency.
- **Alternative explanation:** two alerts for one purchase (authorisation + settlement); an SMS copy of the same transaction (`nova.finance.log-transactions-from-sms`).
- **Contradictory evidence:** a manual entry with a slightly different amount (fees, FX) — no auto-reconcile; flag as possible duplicate in review.
- **Could mislead:** balances that look complete while one institution's alerts are not reaching the mailbox — disclose per-institution last-seen dates.
- **Separate verifier:** the server-side router recomputes the gate; S11 audits scope compliance independently.
- **Findings requiring disclosure:** possible duplicates, unreadable amounts, push failures.

## L. Human escalation

- **Triggers:** review items; expired approval; repeated push failures; external authorization lost; queue backlog > 25 items for 7 days.
- **Why the agent cannot decide:** the account, a duplicate or a backlog item is genuinely uncertain, and a wrong entry corrupts balances.
- **Information presented:** amount + currency, date, counterparty, email subject, reason, suggested account.
- **Decision requested:** approve to an account, or reject; renew the approval; reconnect the app.
- **Options and consequences:** approve logs the item; reject closes it; ignoring leaves it pending (balances may lag).
- **Safe behaviour while waiting:** nothing is logged for that item.
- **If no response:** items stay pending; a weekly reminder (owner channel) until resolved or rejected.
- **Resume:** approval → `nova.finance.review.approve` → record staged → S10 sync → verification.

## M. Completion conditions

Mandatory: `txn_run` valid; every auto-logged record satisfies the scope rules; no duplicate references; skip rules held; no body persisted; sync states truthful; evidence references present.

- `completed` — every selected message reached a definitive outcome and every staged record is `pushed` or safely `pending` with a stated reason.
- `partially_completed` — deferrals, push failures, review notice undelivered, or a sync left `pushing` pending verification.
- `blocked` — invalid config, no authorized mailbox, or no approved LLM provider.
- `failed` — hub store unavailable or a scope/privacy check failed.
- `cancelled` — kill switch or lock held.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome ("1 logged, 1 waiting for review, 1 skipped"); `txn_run`; step statuses S1–S11; verification results and failed checks; evidence references (record IDs, remote IDs, review IDs, verdicts); approvals and scope (approval ID, accounts, caps); side effects (records created, external records created) and reconciliation status; corrections performed; unresolved challenges; limitations (stale mailboxes, unreadable amounts); escalation state; learning candidate. Never include bodies, credentials or full account numbers.

## O. Learning and revision

- **Outcome metrics:** auto-log precision (user reversals), review rate by reason, approve vs reject ratio, duplicate catches, unreadable-amount rate, push failure rate, capture lag.
- **Evidence retained:** run records, verdicts, records, review decisions, per policy.
- **Data minimisation:** no body text; subjects ≤ 200 characters.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}` — for example "approved 10 of 10 items for account X → suggest adding its last-4 hint".
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests plus the fixture precision set.
- **Promotion approval:** publisher review plus Skill Port review; gate changes are versioned and never applied silently.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (fixture card alert, opted-in account) | logged once, pushed with remote ID | UNVERIFIED on Skill Port; observed in the reference implementation |
| Invalid input (bad time zone) / missing input (no approval) | `blocked` / review-only | TBD |
| Missing capability (sync binding absent) | records stay `pending`; `partially_completed` | TBD |
| Permission denied (`finance:hub:records:write` missing) | review-only; nothing logged | TBD |
| Unauthorized scope (account not in approval) | review `account_not_opted_in` | TBD |
| Tenant isolation | no cross-tenant matches or duplicates | TBD |
| Gateway per-order notice | skipped | TBD |
| Store-credit refund | skipped | TBD |
| Manual entry ±3 days | `reconciled`, no new record | TBD |
| Same reference already logged from SMS | `reconciled` | TBD |
| Over cap / medium confidence | review with reason | TBD |
| Backlog message | review `backlog` | TBD |
| Push timeout (ambiguous) | stays `pushing`; verified after 15 min; no double record | TBD |
| Duplicate / replay (same message twice) | one record | TBD |
| False success (200 batch, one item error) | that item `failed`, others `pushed` | TBD |
| Failed verification (scope audit fails) | record moved to review; defect reported | TBD |
| Correction budget exhausted | remaining deferred; safe stop | TBD |
| Escalation (approval expired) | review-only and the user is asked | TBD |
| Cancellation (kill switch) | `cancelled` | TBD |
| Compensation (user reversal) | offsetting entry only with approval | TBD |
| Output schema and status | valid `txn_run` | TBD |
| Compatibility / regression (provider API version) | payload contract test passes | TBD |

All tests use sandbox mailboxes with fictional alerts, a sandbox external-app account and rolled-back transactions; never real accounts or real money. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, approval record format and review UI — TBD (owner: publisher, with Skill Port).
- Default per-transaction caps and approval expiry — TBD (owner: user/operator).
- Alerts whose amount is hidden in images or tracking links cannot be read.
- Fees and FX differences defeat exact-amount duplicate matching; such items go to review as possible duplicates.
- Own-account transfers (one account to another) are logged as an expense plus an income unless the user links them; transfer support is TBD.
- External app rate limits and field mappings are provider-specific (owner: operator).
