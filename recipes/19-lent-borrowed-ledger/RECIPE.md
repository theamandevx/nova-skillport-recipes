# Lent and borrowed ledger

## A. Identity and ownership

- **Recipe ID:** `nova.finance.lent-borrowed-ledger` (proposed until assigned by Skill Port)
- **Name:** Lent and borrowed ledger
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** finance
- **Tags:** ledger, iou, lent, borrowed, receivables, audit-history, confirmation · Vedaxi task type: none (no supported task type covers this today)
- **Summary:** The user tells the agent about money given to or received from people ("I lent Jane 50 USD for dinner"). The agent parses it, shows exactly what it will record, and appends the entry only after the user confirms. It keeps per-person, per-currency balances of who owes whom. Every create, correction and void is written to an audit history, and nothing is ever deleted.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** A conversation channel with the user (chat or voice); a ledger store with append-only entries and an audit table behind dispatcher functions; an LLM for parsing natural language (optional: structured input works without it). All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — natural-language logging, per-person balances (given minus received), confirmation before writes, an audit table and void-instead-of-delete have run in Nova's reference implementation through chat (catalogue status: Built). Not every control described here has evidence yet (Section P lists the tests still needed). Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.finance.ledger` | Capability | Person-to-person money ledger with audit history | medium | UNVERIFIED |
| `nova.finance.ledger.parse` | Operation (read, LLM or form) | Turn a message into a draft entry or a query/correction intent | low | UNVERIFIED |
| `nova.finance.ledger.resolve_counterparty` | Operation (read) | Find the person by name or alias | low | UNVERIFIED |
| `nova.finance.ledger.add_counterparty` | Operation (create, confirmed) | Create a new person record | low | UNVERIFIED |
| `nova.finance.ledger.find_recent_duplicate` | Operation (read) | Look for the same entry recorded recently | low | UNVERIFIED |
| `nova.finance.ledger.append_entry` | Operation (create, confirmed) | Insert one entry and its audit row atomically | medium | UNVERIFIED |
| `nova.finance.ledger.change_entry` | Operation (update + create, confirmed) | Void an entry, or void it and append its replacement, with audit | medium | UNVERIFIED |
| `nova.finance.ledger.balance` | Operation (read) | Per-person, per-currency balances from non-voided entries | low | UNVERIFIED |
| `nova.finance.ledger.history` | Operation (read) | Entries and the full audit trail for a person or entry | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.finance.ledger.parse.llm` — LLM parsing of free text into `draft-entry@0.1.0`, temperature 0. Selected for chat and voice input.
- `nova.finance.ledger.parse.form` — structured fields from a form or command (`/ledger gave Jane 50 USD dinner`). No LLM. Selected when the input is already structured. Both variants are read-only and produce the same schema; neither writes anything.
- `nova.finance.ledger.append_entry.default` — security-definer database function: entry insert plus audit insert in one transaction.
- `nova.finance.ledger.change_entry.void` — sets `status = voided` with a reason; audit row with before/after.
- `nova.finance.ledger.change_entry.correct` — voids the original and appends a replacement linked by `replaces_entry_id`, in one transaction; two audit rows.

**Bindings:** `binding://nova/finance/ledger-parse`, `binding://nova/finance/ledger-counterparty`, `binding://nova/finance/ledger-duplicate`, `binding://nova/finance/ledger-append`, `binding://nova/finance/ledger-change`, `binding://nova/finance/ledger-balance`, `binding://nova/finance/ledger-history` — all `status: UNVERIFIED`. Reference implementation: an assistant skill invoked from chat calls a webhook gateway on the automation hub, which calls ledger dispatcher functions in PostgreSQL (entries table plus an audit table; void sets a flag that excludes the entry from balances).

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Counterparty | `schema://nova/lent-borrowed-ledger/counterparty@0.1.0` | the requesting user | read; create (confirmed); update (aliases, confirmed) |
| Entry | `schema://nova/lent-borrowed-ledger/entry@0.1.0` | the requesting user | create (confirmed); update (`status`, `voided_at`, `void_reason` only, confirmed) |
| Audit row | `schema://nova/lent-borrowed-ledger/audit-row@0.1.0` | the requesting user | create (system, with every change); read |
| Balance | `schema://nova/lent-borrowed-ledger/balance@0.1.0` | derived | read |

**Authorization requirements (declared, never a grant):** `finance:ledger:read`, `finance:ledger:write`, `llm:parse` (LLM variant only). Database role: `secret_ref://finance-db-role`; LLM key: `secret_ref://llm`.

**Policy / trust:** risk class `medium` (personal financial records about third parties; no money movement). Every write needs the user's confirmation of the exact entry. Counterparty names are personal data and stay within the user's tenant.

## C. Intent and applicability

- **User problem:** People forget who paid for what, small loans go uncollected, and balances get disputed because nobody kept a clear history.
- **Desired outcome:** An accurate running balance per person and currency, with a full, tamper-evident history of every entry and change.
- **Trigger or entry conditions:** The user says they gave, lent, paid for, borrowed, received or were repaid money with a person; asks "who owes me?" or "what does Jane owe?"; or asks to fix or remove an entry.
- **Applicable situations:** Personal IOUs, split bills, small loans, repayments, money held for someone.
- **Out-of-scope situations:** Personal expenses or income with no counterparty (that belongs to the finance hub); requesting or sending payments; reminding the counterparty; interest calculations; business invoicing; deleting history.
- **Required inputs:** `user_message` (see Section D).
- **Prerequisites:** The ledger store installed; the conversation channel identifies the user.
- **Required capabilities:** `nova.finance.ledger` (proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port; every binding is a dependency to resolve before activation.
- **Assumptions:** Verified in the reference implementation: the gave/received model, balances as given minus received per person, confirmation before writes, audit rows, and void instead of delete. Not every control described here has evidence yet (Section P lists the tests still needed).
- **Ambiguity policy:** Ask. If the person, direction, amount or currency is unclear, ask one question; never assume a direction ("paid" alone is ambiguous: paid *for* them or paid them *back*).
- **Vedaxi task type:** none.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `user_message` | text ≤ 1,000 chars, or structured form fields | yes | the user; personal, financial | — | Ask what to record. |
| `default_currency` | ISO 4217 | no | user setting | none | No default and no currency stated → ask. A default is always shown in the confirmation. |
| `timezone` | IANA zone | yes | profile | — | Ask once, or use the runtime default and show the resolved date in the confirmation. |
| `channel_ref` | conversation ref | yes | runtime | — | Cannot confirm → `blocked`. |

```json
{"$id":"schema://nova/lent-borrowed-ledger/ledger-request@0.1.0","type":"object","required":["user_message","channel_ref"],
 "properties":{"user_message":{"type":"string","minLength":2,"maxLength":1000},"form":{"type":["object","null"]},
 "default_currency":{"type":["string","null"],"pattern":"^[A-Z]{3}$"},"timezone":{"type":"string"},"channel_ref":{"type":"string"}}}
{"$id":"schema://nova/lent-borrowed-ledger/parse-result@0.1.0","type":"object","required":["intent"],
 "properties":{"intent":{"enum":["record","balance","history","void","correct","not_ledger","unclear"]},
 "draft":{"anyOf":[{"$ref":"schema://nova/lent-borrowed-ledger/draft-entry@0.1.0"},{"type":"null"}]},
 "target_entry_id":{"type":["string","null"]},"counterparty_name":{"type":["string","null"]},"question":{"type":["string","null"],"maxLength":200}}}
{"$id":"schema://nova/lent-borrowed-ledger/draft-entry@0.1.0","type":"object","required":["counterparty_name","direction","amount","currency","entry_date"],
 "properties":{"counterparty_name":{"type":"string","minLength":1,"maxLength":80},"direction":{"enum":["gave","received"]},
 "amount":{"type":"string","pattern":"^(?!0+(\\.0+)?$)[0-9]{1,13}(\\.[0-9]{1,4})?$"},"currency":{"type":"string","pattern":"^[A-Z]{3}$"},
 "currency_source":{"enum":["stated","default_setting"]},"entry_date":{"type":"string","format":"date"},"reason":{"type":["string","null"],"maxLength":200}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `ledger_result` | `schema://nova/lent-borrowed-ledger/ledger-result@0.1.0` | `status`, `action` | reply in the same conversation; run record | personal, financial; owner only | read-back + independent balance recomputation (Section I) |

```json
{"$id":"schema://nova/lent-borrowed-ledger/counterparty@0.1.0","type":"object","required":["counterparty_id","display_name"],
 "properties":{"counterparty_id":{"type":"string"},"display_name":{"type":"string","maxLength":80},"aliases":{"type":"array","items":{"type":"string","maxLength":80}},"created_at":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/lent-borrowed-ledger/counterparty-query@0.1.0","type":"object","required":["name"],"properties":{"name":{"type":"string","minLength":1,"maxLength":80}}}
{"$id":"schema://nova/lent-borrowed-ledger/counterparty-match@0.1.0","type":"object","required":["result"],"properties":{"result":{"enum":["exact","candidates","none"]},"counterparty":{"anyOf":[{"$ref":"schema://nova/lent-borrowed-ledger/counterparty@0.1.0"},{"type":"null"}]},"candidates":{"type":"array","items":{"$ref":"schema://nova/lent-borrowed-ledger/counterparty@0.1.0"}}}}
{"$id":"schema://nova/lent-borrowed-ledger/entry@0.1.0","type":"object",
 "required":["entry_id","counterparty_id","direction","amount","currency","entry_date","status","confirmation_ref","created_at"],
 "properties":{"entry_id":{"type":"string"},"counterparty_id":{"type":"string"},"direction":{"enum":["gave","received"]},
 "amount":{"type":"string","pattern":"^[0-9]{1,13}(\\.[0-9]{1,4})?$"},"currency":{"type":"string","pattern":"^[A-Z]{3}$"},
 "entry_date":{"type":"string","format":"date"},"reason":{"type":["string","null"],"maxLength":200},
 "status":{"enum":["active","voided"]},"voided_at":{"type":["string","null"],"format":"date-time"},"void_reason":{"type":["string","null"],"maxLength":200},
 "replaces_entry_id":{"type":["string","null"]},"confirmation_ref":{"type":"string"},"source_message_ref":{"type":["string","null"]},"created_at":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/lent-borrowed-ledger/audit-row@0.1.0","type":"object","required":["audit_id","action","actor","at","after"],
 "properties":{"audit_id":{"type":"string"},"entry_id":{"type":["string","null"]},"counterparty_id":{"type":["string","null"]},
 "action":{"enum":["entry_create","entry_void","entry_correct","counterparty_create","counterparty_alias_add"]},
 "actor":{"type":"string"},"channel":{"type":"string"},"at":{"type":"string","format":"date-time"},
 "before":{"type":["object","null"]},"after":{"type":"object"},"reason":{"type":["string","null"],"maxLength":200},"confirmation_ref":{"type":["string","null"]}}}
{"$id":"schema://nova/lent-borrowed-ledger/dup-check@0.1.0","type":"object","required":["result"],"properties":{"result":{"enum":["none","possible"]},"entry_ids":{"type":"array","items":{"type":"string"}}}}
{"$id":"schema://nova/lent-borrowed-ledger/change-request@0.1.0","type":"object","required":["entry_id","kind","reason","confirmation_ref"],
 "properties":{"entry_id":{"type":"string"},"kind":{"enum":["void","correct"]},"reason":{"type":"string","minLength":2,"maxLength":200},
 "replacement":{"anyOf":[{"$ref":"schema://nova/lent-borrowed-ledger/draft-entry@0.1.0"},{"type":"null"}]},"confirmation_ref":{"type":"string"}}}
{"$id":"schema://nova/lent-borrowed-ledger/balance@0.1.0","type":"object","required":["counterparty_id","display_name","by_currency"],
 "properties":{"counterparty_id":{"type":"string"},"display_name":{"type":"string"},
 "by_currency":{"type":"object","additionalProperties":{"type":"string","pattern":"^-?[0-9]{1,13}(\\.[0-9]{1,4})?$"},"description":"sum(gave) - sum(received) over active entries; > 0 they owe the user; < 0 the user owes them"},
 "entry_count":{"type":"integer","minimum":0},"last_entry_date":{"type":["string","null"],"format":"date"}}}
{"$id":"schema://nova/lent-borrowed-ledger/history-query@0.1.0","type":"object","properties":{"counterparty_id":{"type":["string","null"]},"entry_id":{"type":["string","null"]},"include_voided":{"type":"boolean"},"limit":{"type":"integer","minimum":1,"maximum":500}}}
{"$id":"schema://nova/lent-borrowed-ledger/history@0.1.0","type":"object","required":["entries","audit"],"properties":{"entries":{"type":"array","items":{"$ref":"schema://nova/lent-borrowed-ledger/entry@0.1.0"}},"audit":{"type":"array","items":{"$ref":"schema://nova/lent-borrowed-ledger/audit-row@0.1.0"}}}}
{"$id":"schema://nova/lent-borrowed-ledger/ledger-result@0.1.0","type":"object","required":["status","action"],
 "properties":{"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},
 "action":{"enum":["recorded","voided","corrected","balance","history","none"]},"entry_id":{"type":["string","null"]},
 "audit_ids":{"type":"array","items":{"type":"string"}},"balances":{"type":"array","items":{"$ref":"schema://nova/lent-borrowed-ledger/balance@0.1.0"}},
 "message":{"type":"string","maxLength":600}}}
```

**Example (fictional; today 2026-11-03, America/New_York, default currency USD)**

Input: "Lent Jane 50 for dinner yesterday."

Confirmation shown: "Record: you **gave** Jane Doe **50.00 USD** (your default currency) on **Mon 2 Nov**, reason 'dinner'? Jane's balance would go from owes you 70.00 USD to owes you 120.00 USD. [Confirm] [Edit] [Cancel]"

Output after confirm: `{"status":"completed","action":"recorded","entry_id":"led_0031","audit_ids":["aud_0102"],"balances":[{"counterparty_id":"cp_007","display_name":"Jane Doe","by_currency":{"USD":"120.00"},"entry_count":5,"last_entry_date":"2026-11-02"}],"message":"Saved. Jane Doe owes you 120.00 USD."}`

## E. Constraints and budgets

- **Hard constraints:** every entry, correction and void is confirmed by the user on the exact values shown; never delete entries, audit rows or counterparties; corrections void and replace, never edit amounts in place; amounts are positive decimal strings and the currency is always explicit (a default currency is shown and confirmed); balances are per currency and never netted across currencies; never move money, send payment requests or contact the counterparty; personal expenses without a counterparty are not ledger entries.
- **Soft constraints:** keep reasons short; use the person's existing display name; show the new balance in every confirmation.
- **Resource and tenant scope:** the requesting user's ledger only.
- **Data sources allowed:** the user's message and the user's ledger.
- **External transfers permitted:** the message text to the approved LLM provider (LLM variant only).
- **Time limit:** 30 seconds per step, excluding the wait for the user's confirmation.
- **Cost or token budget:** at most 2 LLM calls per request (parse plus one re-parse after an edit).
- **Maximum steps:** 10 (S1–S10); one request uses a subset.
- **Maximum retries and correction cycles:** reads 2; writes 2 after read-back; edits at confirmation: 3 rounds, then cancel.
- **Rate limits and concurrency:** writes serialised per user.
- **Retention and logging constraints:** entries and audit kept until the user closes the ledger (retention per operator policy, `TBD`); logs hold IDs, not names or amounts.
- **Prohibited actions:** writing without confirmation; inferring direction from tone; merging two people without confirmation; hiding voided entries from the history.
- **Stop conditions:** the user cancels; confirmation times out (30 minutes); store unavailable.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Entry + audit row | validated draft; counterparty resolved | Approval Required (S5, per entry) | read-back; balance recomputed |
| Create | Counterparty + audit row | no existing match; the user confirms the new person | Approval Required | read-back |
| Read | Entries, counterparties, audit, balances | own tenant | Allowed | — |
| Update | Entry `status`, `voided_at`, `void_reason` (void) | entry active | Approval Required | audit row with before/after |
| Update | Counterparty `aliases` (add only) | the user confirms "Jen = Jane Doe" | Approval Required | audit row |
| Delete | entries, audit rows, counterparties | — | Not Allowed | — |
| External side effect | none beyond replies to the user | — | — | — |

- **Idempotency:** each confirmation has a `confirmation_ref`; `append_entry` and `change_entry` are unique on it, so a retried write cannot create a second entry.
- **Expected side effects:** entry and audit rows; changed balances.
- **Compensation:** void (or void and replace) with a new confirmation; the original stays in the history.
- **Irreversible:** none within the ledger (everything is append-only and reversible by a voiding entry); parse text already sent to the LLM.
- **Ambiguous outcomes:** if a write times out, read by `confirmation_ref` before retrying.

## G. Autonomy and approval policy

- **Default autonomy level:** A1 — supervised execution.
- **Planning level:** A2 (parse, resolve, check duplicates).
- **Execution level:** A1 — writes only after the S5 confirmation.
- **Verification level:** A2.
- **Recovery level:** A1.
- **Approval required for:** every entry; every void or correction; creating a counterparty; adding an alias.
- **Actions never permitted:** deleting anything; editing amounts in place; logging without confirmation; contacting counterparties; moving money.
- **Conditions requiring clarification:** an unknown or ambiguous person; an ambiguous direction; a missing amount or currency; a date more than 1 year in the past or any date in the future.
- **Conditions requiring escalation:** a requested change to an entry older than 12 months, or to an entry that a later settlement depends on (show the effect on the balance first).
- **Safe state after stop, cancellation or escalation:** nothing written; the draft discarded.

Confirmation covers exactly the entry shown. It does not cover later entries, other people or edits.

## H. Execution plan

**S1 — Parse the request**
1. Objective: understand what the user wants.
2. Capability: `nova.finance.ledger.parse` (`llm` for free text, `form` for structured input).
3. Input mapping: `user_message`, `default_currency`, today's date in `timezone` → `parse-result@0.1.0`.
4. Dependencies: none.
5. Preconditions: user identified.
6. Action / side effects: one LLM call (LLM variant) or field mapping; no writes.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 1.
9. Idempotency: no state change.
10. Postconditions: an intent and, for `record`, a draft.
11. Verification: schema valid; a `record` draft has all required fields or an explicit question.
12. Failure routes: `not_ledger` (an expense with no person) → explain and suggest the finance hub; `unclear` → ask the question; `balance` or `history` → S10; `void` or `correct` → S9.
13. Escalation triggers: none.
14. Evidence: intent, parse variant.

**S2 — Validate the draft**
1. Objective: no ambiguous money values.
2. Capability: deterministic checks.
3. Input mapping: draft.
4. Dependencies: S1.
5. Preconditions: intent `record`.
6. Action: amount > 0 as a decimal string; currency stated or from the default setting (marked as such); direction explicit; date not in the future and ≤ 1 year old.
7. Autonomy: A2.
8. Timeout / retry: 1 s / none.
9. Idempotency: pure.
10. Postconditions: valid draft.
11. Verification: `draft-entry@0.1.0` valid.
12. Failure routes: any missing or ambiguous field → ask one question → re-run S1 with the answer (max 3 rounds) → else `cancelled`.
13. Escalation: none.
14. Evidence: validation result.

**S3 — Resolve the counterparty**
1. Objective: exactly one person.
2. Capability: `nova.finance.ledger.resolve_counterparty`; `add_counterparty` if new.
3. Input mapping: `counterparty_name`.
4. Dependencies: S2.
5. Preconditions: none.
6. Action / side effects: read; exact display-name or alias match → use it; several candidates → ask which; none → propose "Add Jane Doe as a new person?" (created only at S6 after confirmation).
7. Autonomy: A2 read; A0 choice.
8. Timeout / retry: 5 s / 2.
9. Idempotency: read-only.
10. Postconditions: a counterparty ID or a confirmed new name.
11. Verification: the counterparty belongs to the user.
12. Failure routes: the user cannot say who → `cancelled`.
13. Escalation: none.
14. Evidence: match result.

**S4 — Duplicate guard**
1. Objective: avoid recording the same thing twice.
2. Capability: `nova.finance.ledger.find_recent_duplicate`.
3. Input mapping: counterparty, direction, amount, currency.
4. Dependencies: S3.
5. Preconditions: existing counterparty.
6. Action / side effects: read; the same values recorded in the last 10 minutes, or the same source message already recorded → `possible`.
7. Autonomy: A2.
8. Timeout / retry: 5 s / 2.
9. Idempotency: read-only.
10. Postconditions: `none` or `possible`.
11. Verification: matches belong to the user.
12. Failure routes: `possible` → the confirmation says "You recorded the same 2 minutes ago — record again?"
13. Escalation: none.
14. Evidence: matched entry IDs.

**S5 — Confirmation checkpoint**
1. Objective: the user approves the exact entry.
2. Capability: user interaction in the conversation.
3. Input mapping: draft, counterparty (new or existing), current and projected balance, duplicate notice.
4. Dependencies: S4.
5. Preconditions: valid draft.
6. Action: show direction in plain words ("you gave" / "you received"), amount, currency (and that it came from the default if so), date, reason, the person, and the balance before and after. Options: Confirm / Edit / Cancel.
7. Autonomy: A0 — human decision.
8. Timeout / retry: wait 30 minutes, then `cancelled`.
9. Idempotency: a new `confirmation_ref` per shown draft.
10. Postconditions: a confirmation record bound to the draft hash.
11. Verification: the confirmation references this draft hash.
12. Failure routes: Edit → S1 with the edit; Cancel or timeout → `cancelled`.
13. Escalation: not applicable.
14. Evidence: confirmation ref, timestamp, draft hash.

**S6 — Append the entry**
1. Objective: write exactly the confirmed entry.
2. Capability: `nova.finance.ledger.append_entry.default` (plus `add_counterparty` in the same transaction if new).
3. Input mapping: confirmed draft, counterparty, `confirmation_ref`.
4. Dependencies: S5.
5. Preconditions: confirmation valid and unexpired.
6. Action / side effects: insert the entry (and counterparty) plus audit rows atomically.
7. Autonomy: A1, covered by S5.
8. Timeout / retry: 10 s / 2 after read-back by `confirmation_ref`.
9. Idempotency: unique `confirmation_ref`.
10. Postconditions: one active entry; audit rows.
11. Verification: returned IDs.
12. Failure routes: store error → `failed`; nothing written (atomic).
13. Escalation: none.
14. Evidence: entry ID, audit IDs.

**S7 — Verify**
1. Objective: the stored entry and balance are right.
2. Capability: `nova.finance.ledger.history`, `nova.finance.ledger.balance`.
3. Input mapping: entry ID, counterparty ID.
4. Dependencies: S6.
5. Preconditions: write returned.
6. Action: read the entry; recompute the balance independently from the full active-entry history; compare with `balance`.
7. Autonomy: A2.
8. Timeout / retry: 5 s / 1.
9. Idempotency: read-only.
10. Postconditions: verified.
11. Verification: stored values equal the confirmed draft; recomputed balance = the balance shown in S5 as "after"; an audit row exists.
12. Failure routes: mismatch → void the new entry with reason `verification_failed` (audited), tell the user, `failed`.
13. Escalation: repeated mismatch → disable writes for the user until the operator reviews.
14. Evidence: verification result.

**S8 — Reply**
1. Objective: tell the user it is saved and where things stand.
2. Capability: reply in the conversation.
3. Input mapping: `ledger_result`.
4. Dependencies: S7.
5. Preconditions: verified.
6. Action / side effects: "Saved. Jane Doe owes you 120.00 USD."
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2.
9. Idempotency: reply key = `confirmation_ref`.
10. Postconditions: reply delivered.
11. Verification: the delivery status of the reply.
12. Failure routes: reply not delivered → the entry stays recorded; `partially_completed`; shown in the history next time.
13. Escalation: none.
14. Evidence: reply ID.

**S9 — Void or correct an entry (on request)**
1. Objective: fix a mistake without losing history.
2. Capability: `nova.finance.ledger.change_entry` (`void` or `correct`).
3. Input mapping: target entry (from S1 or picked from the history), reason, replacement draft (for a correction, validated as in S2).
4. Dependencies: S1.
5. Preconditions: entry active and owned by the user.
6. Action / side effects: show the entry, the change and the balance effect → the user confirms → void (or void plus replacement) and audit rows in one transaction.
7. Autonomy: A0 for the decision; A1 for the write.
8. Timeout / retry: confirmation 30 minutes; write 10 s / 2 after read-back.
9. Idempotency: unique `confirmation_ref`.
10. Postconditions: original `voided`; replacement active (for corrections); audit rows.
11. Verification: as in S7, including the history still showing the voided entry.
12. Failure routes: entry already voided → tell the user; cancelled → nothing changed.
13. Escalation: entries older than 12 months → show the effect and ask for an explicit second confirmation.
14. Evidence: audit IDs.

**S10 — Answer a balance or history question (read-only)**
1. Objective: who owes what, with the proof.
2. Capability: `nova.finance.ledger.balance`, `nova.finance.ledger.history`.
3. Input mapping: optional counterparty, `include_voided`.
4. Dependencies: S1.
5. Preconditions: none.
6. Action / side effects: read; reply with per-currency balances ("Jane owes you 120.00 USD and 15.00 EUR; you owe John Roe 40.00 USD"), with recent entries on request.
7. Autonomy: A2.
8. Timeout / retry: 5 s / 2.
9. Idempotency: read-only.
10. Postconditions: answer delivered.
11. Verification: balances recomputed from entries before replying.
12. Failure routes: store down → `failed` with an apology; no guessed numbers.
13. Escalation: none.
14. Evidence: query and result counts.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Input validity | draft valid; direction and currency explicit | validator | valid | ask |
| Confirmation scope | stored entry = the confirmed draft (hash) | S6 vs S5 | equal | void with audit; `failed` |
| Append-only | no entry or audit row deleted; amounts never edited in place | store audit | holds | stop; escalate |
| Audit completeness | every change has an audit row with before/after | S7/S9 | holds | `partially_completed`; report |
| Balance truth | recomputed balance = function balance | S7, S10 | equal | stop writes; escalate |
| Currency separation | no cross-currency netting | balance output | holds | fix output |
| Tenant | all rows belong to the user | read-back | holds | stop |
| Output schema | `ledger_result` valid | validator | valid | fix; never fake |

## J. Self-correction and recovery

- **Failure classes:** parse failure, missing field, ambiguous person, duplicate suspicion, confirmation timeout, store failure, verification mismatch.
- **Permitted corrections:** ask a clarifying question; re-parse after an edit; read back before retrying a write; void a write that failed verification (audited).
- **Forbidden corrections:** assuming a direction, currency or person; writing without confirmation; deleting a bad entry; editing an amount in place.
- **Maximum correction cycles:** 3 clarification rounds; 1 write retry after read-back.
- **Reverification required after correction:** yes — S7.
- **Stop conditions:** the user cancels; timeout; store unavailable.
- **Recovery or compensation:** void or void-and-replace with audit.
- **What must be reported:** what was written, voided or corrected, and the resulting balances.

## K. Independent challenge

- **Assumptions to challenge:** "paid" means paid *for* them (gave) rather than paid them *back* (also gave, but settling a negative balance) — both are `gave`, but the confirmation shows the balance effect so the user can catch a misunderstanding; "Jen" is Jane Doe.
- **Alternative explanation:** the money was a gift (not a loan) — the confirmation offers "don't record".
- **Contradictory evidence:** a balance flipping sign unexpectedly (for example the user now "owes" someone after recording what they thought was a loan) — highlighted in the confirmation.
- **Could mislead:** a single-currency summary that hides a debt in another currency — balances are always listed per currency.
- **Separate verifier:** S7 recomputes balances from the raw history independently of the balance function.
- **Findings requiring disclosure:** sign flips, default-currency use, possible duplicates.

## L. Human escalation

- **Triggers:** an ambiguous person, direction or currency; possible duplicates; changes to entries older than 12 months; repeated verification mismatch.
- **Why the agent cannot decide:** only the user knows who and what is meant; old changes affect settled relationships.
- **Information presented:** the draft or target entry, the person, the balance before and after, the reason.
- **Decision requested:** confirm, edit or cancel.
- **Options and consequences:** confirming writes the entry and changes the balance; cancelling leaves everything unchanged.
- **Safe behaviour while waiting:** nothing written.
- **If no response:** after 30 minutes `cancelled`; the draft is discarded.
- **Resume:** the user repeats the request; a new confirmation is required.

## M. Completion conditions

Mandatory: `ledger_result` valid; any write matches its confirmation, has audit rows and passed balance verification; nothing deleted; balances per currency; evidence references present.

- `completed` — the entry, void or correction was confirmed, written and verified, and the reply delivered; or a query was answered.
- `partially_completed` — written and verified, but the reply could not be delivered.
- `blocked` — no conversation channel to confirm in, or no ledger store binding.
- `failed` — store error, or verification failed (and the new entry was voided).
- `cancelled` — the user cancelled, did not confirm within 30 minutes, or exhausted the clarification rounds.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome ("Saved: you gave Jane Doe 50.00 USD; she owes you 120.00 USD"); `ledger_result`; step statuses for the steps used; verification results; evidence references (entry ID, audit IDs, confirmation ref); approvals and scope (the confirmation); side effects (rows written) and reconciliation status; corrections performed; unresolved challenges; limitations; escalation state; learning candidate. Never expose other users' data or include credentials.

## O. Learning and revision

- **Outcome metrics:** clarification rate by field, edit-at-confirmation rate, void and correction rate, duplicate-guard hits, verification failures.
- **Evidence retained:** entries and audit history (the user's own records), run records.
- **Data minimisation:** run records hold IDs only; message text is not stored beyond the parse.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}` — for example a phrasing that is often misparsed for direction.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests plus the direction-phrasing fixture set.
- **Promotion approval:** publisher review plus Skill Port review.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path ("lent Jane 50 USD for dinner" → confirm) | entry, audit row, correct balance | UNVERIFIED on Skill Port; observed in the reference implementation |
| Invalid input (amount "abc" / negative) | asks again; nothing written | TBD |
| Missing input (no currency, no default) | asks for the currency | TBD |
| Ambiguous direction ("paid Jane 20") | asks gave or received | TBD |
| New person | confirmation includes "add new person" | TBD |
| Missing capability (append binding absent) | `blocked`; nothing written | TBD |
| Permission denied (`finance:ledger:write` missing) | read-only answers; writes `blocked` | TBD |
| Unauthorized scope (another user's entry ID in a void request) | refused | TBD |
| Tenant isolation | balances never include other users' entries | TBD |
| Expense with no person | `not_ledger`; finance hub suggested | TBD |
| Possible duplicate within 10 minutes | warning in the confirmation | TBD |
| Timeout on write | read-back by `confirmation_ref`; no double entry | TBD |
| Duplicate / replay (same confirmation twice) | one entry | TBD |
| False success (write OK, read-back differs) | entry voided with audit; `failed` | TBD |
| Failed verification (balance mismatch) | writes disabled; escalation | TBD |
| Correction budget exhausted (3 clarification rounds) | `cancelled` | TBD |
| Escalation (change to a 2-year-old entry) | second confirmation required | TBD |
| Cancellation (user cancels at S5) | nothing written | TBD |
| Compensation (void, then correct) | history shows original, void and replacement; balance right | TBD |
| Multi-currency balance | per-currency lines, no netting | TBD |
| Output schema and status | valid result | TBD |
| Compatibility / regression (new parser model) | direction fixtures pass | TBD |

All tests use fictional people and amounts in a sandbox ledger or rolled-back transactions; never real ledgers. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, the confirmation UI and the ledger viewer — TBD (owner: publisher, with Skill Port).
- Mirroring receivables into an external personal-finance app is out of scope for 0.1.0; a separate recipe is needed (owner: publisher).
- Shared expenses split among several people need one entry per person; a split helper is TBD.
- No reminders are sent to counterparties by design; a "nudge me to collect" reminder to the user is a possible later feature.
- Retention policy for closed ledgers — TBD (owner: operator).
