# Auto-mark bills paid

## A. Identity and ownership

- **Recipe ID:** `nova.finance.auto-mark-bills-paid` (proposed until assigned by Skill Port)
- **Name:** Auto-mark bills paid
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** finance
- **Tags:** payments, reconciliation, receipts, invoice-matching, payments-log, paid-tracking · Vedaxi task type: none (no supported task type covers this today)
- **Summary:** When a receipt, tax invoice or "payment received / renewed" message is detected (or the user says a bill is paid), the agent records it in a payments log. It then reconciles the payment against open obligations: by invoice or order number first, otherwise by product, amount, currency and timing. A matched one-off bill is marked paid; a matched recurring bill moves to its next cycle. This is alert and paid-tracking only: it never changes account balances and never logs transactions.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** The obligation store and the payment-confirmation hand-off from `nova.finance.upcoming-payments-radar`; a payments-log store; optionally the owner notification channel. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — reference-number matching, series + amount + timing matching, cycle advance, prior-cycle rejection, idempotent re-delivery and presumed-paid closing of auto-charge one-offs have run in Nova's reference implementation, tested with rolled-back transactions and live messages (catalogue status: Built). Not every control described here has evidence yet (Section P lists the tests still needed). Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.finance.payments` | Capability | The payments log and reconciliation against obligations | medium | UNVERIFIED |
| `nova.finance.payments.get_by_source` | Operation (read) | Return the log row for a source reference, if any | low | UNVERIFIED |
| `nova.finance.payments.match` | Operation (read) | Find candidate open obligations for a payment event and return a decision | low | UNVERIFIED |
| `nova.finance.payments.commit` | Operation (create + update) | In one transaction: insert the log row and settle the matched obligation (or log only) | medium | UNVERIFIED |
| `nova.finance.payments.link` | Operation (update, user-approved) | Link an ambiguous or unmatched log row to an obligation the user picks | medium | UNVERIFIED |
| `nova.finance.obligations` | Capability | The user's payment schedule | medium | UNVERIFIED |
| `nova.finance.obligations.presumed_paid_sweep` | Operation (update) | Close one-off auto-charge obligations past due as *presumed* paid | low | UNVERIFIED |
| `nova.finance.obligations.list` | Operation (read) | Read obligations for verification | low | UNVERIFIED |
| `nova.notify.owner` | Capability | Deliver a message to the requesting user's own channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Ask the user to resolve an ambiguous match | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.finance.payments.match.user-stated` — the user names or selects the obligation ("mark the Example Power & Water bill paid"). No inference is involved. Selected first when present.
- `nova.finance.payments.match.ref` — exact match of the normalised invoice or order number (trimmed, upper-cased, spaces and `#` removed) against an **open** obligation's `external_ref`. A reference match wins even when the series key differs.
- `nova.finance.payments.match.series-amount-timing` — eligible only when `.ref` finds nothing. Requires `series_key` equal, currency equal, amount within ± `amount_tolerance_pct` (default 12 %) of the obligation amount, and `paid_date` inside [`due_date` − 5 days, `due_date` + 45 days]. Exactly one candidate must qualify. The timing gate stops a receipt for the previous cycle from closing the next one.
- All three match variants are read-only, so moving down the order never introduces a side effect.
- `nova.finance.payments.commit.default` — security-definer database function, atomic, conflict-safe on `source_ref`.
- `nova.finance.obligations.presumed_paid_sweep.default` — one-off or unknown-recurrence obligations with `auto_charge = true`, at least `grace_days` past due, with no failed-payment notice on record. They are closed with `paid_basis = presumed_auto_charge` and a log row whose source is `auto:<obligation_id>`. Enabled by default and configurable. Results are always labelled "presumed", never "confirmed".

**Bindings:** `binding://nova/finance/payments-get`, `binding://nova/finance/payments-match`, `binding://nova/finance/payments-commit`, `binding://nova/finance/payments-link`, `binding://nova/finance/obligations-presumed-sweep`, `binding://nova/finance/obligations-list`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: a payment branch inside the radar's apply function (PostgreSQL, security-definer) plus a maintenance step in the reminder scan.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Payment event | `schema://nova/auto-mark-bills-paid/payment-event@0.1.0` | the requesting user | read (input) |
| Payments-log row | `schema://nova/auto-mark-bills-paid/payment-log@0.1.0` | the requesting user's tenant | create; update (`matched_obligation_id`, `match_status` via `link` only) |
| Obligation | `schema://nova/auto-mark-bills-paid/obligation-settle-view@0.1.0` | the requesting user's tenant | read; update (`status`, `paid_at`, `paid_basis`, `due_date` +1 period, cycle flags, `last_paid_at`) |

**Authorization requirements (declared, never a grant):** `finance:payments:read`, `finance:payments:write`, `finance:obligations:read`, `finance:obligations:settle`, `notify:owner`. Database role: `secret_ref://finance-db-role`.

**Policy / trust:** risk class `medium` (a wrong match hides a real unpaid bill). The recipe has no finance-hub scope at all, so it cannot change a balance even by mistake.

## C. Intent and applicability

- **User problem:** After paying, the user still sees the bill as due, gets reminded again, and has to mark things paid by hand.
- **Desired outcome:** Paid bills close themselves, recurring bills move to their next cycle, and every detected payment is in a log, with honest labels for anything inferred.
- **Trigger or entry conditions:** A `payment_confirmation` hand-off from the radar; the user saying a specific bill is paid; or the scheduled presumed-paid sweep (runs with the reminder schedule).
- **Applicable situations:** Receipts, tax invoices, "payment received", "your subscription was renewed on …", card-statement payment acknowledgements.
- **Out-of-scope situations:** Logging the money movement (that is `nova.finance.log-transactions-from-email` / `-from-sms`); changing balances; refunds; money the user received (incoming revenue notices); paying anything.
- **Required inputs:** `payment_event` (see Section D).
- **Prerequisites:** Obligation store installed; payments-log store installed; for email-derived events, the radar installed.
- **Required capabilities:** `nova.finance.payments`, `nova.finance.obligations` (proposed, UNVERIFIED); `nova.notify.owner` (optional).
- **Unsupported or missing capabilities:** None registered on Skill Port; every binding is a dependency to resolve before activation.
- **Assumptions:** Verified in the reference implementation: reference-first matching, the ±12 % / [−5, +45] day fallback, the one-off-paid vs recurring-advance split, idempotency on the source message, and incoming-revenue exclusion. Not every control described here has evidence yet (Section P lists the tests still needed).
- **Ambiguity policy:** Ask. More than one candidate → log as `ambiguous` and ask the user which bill it settles; never pick one.
- **Vedaxi task type:** none.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `payment_event` | `payment-event@0.1.0` | yes | radar hand-off or the user; financial | — | Invalid → `blocked`, nothing logged. |
| `amount_tolerance_pct` | number 0–25 | no | operator | 12 | Clamp. |
| `timing_window_days` | `{before:0–15, after:0–90}` | no | operator | `{5,45}` | Clamp. |
| `presumed_paid` | `{enabled:boolean, grace_days:0–14}` | no | user | `{true,2}` | — |
| `timezone` | IANA zone | yes | user profile | — | `blocked`. |

```json
{"$id":"schema://nova/auto-mark-bills-paid/payment-event@0.1.0","type":"object",
 "required":["source_ref","basis","direction","merchant","series_key","paid_date"],
 "properties":{"source_ref":{"type":"string","pattern":"^(msg|user|auto):[A-Za-z0-9._-]{4,80}$"},
 "basis":{"enum":["receipt","user_stated","presumed_auto_charge"]},"direction":{"enum":["pay","receive"]},
 "merchant":{"type":"string","maxLength":120},"series_key":{"type":"string","pattern":"^[a-z0-9][a-z0-9-]{2,79}$"},
 "amount":{"type":["string","null"],"pattern":"^[0-9]{1,13}(\\.[0-9]{1,4})?$"},"currency":{"type":["string","null"],"pattern":"^[A-Z]{3}$"},
 "paid_date":{"type":"string","format":"date"},"external_ref":{"type":["string","null"],"maxLength":64},
 "obligation_id":{"type":["string","null"],"description":"set only for basis=user_stated"},
 "confidence":{"enum":["high","medium","low"]}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `reconcile_result` | `schema://nova/auto-mark-bills-paid/reconcile-result@0.1.0` | `payment_id`, `outcome`, `match_status` | run record; "Recently paid" view | financial; owner only | read-back of the log row and the obligation (Section I) |

```json
{"$id":"schema://nova/auto-mark-bills-paid/payment-log@0.1.0","type":"object",
 "required":["payment_id","source_ref","series_key","merchant","paid_date","basis","match_status"],
 "properties":{"payment_id":{"type":"string"},"source_ref":{"type":"string"},"series_key":{"type":"string"},"merchant":{"type":"string"},
 "amount":{"type":["string","null"],"pattern":"^[0-9]{1,13}(\\.[0-9]{1,4})?$"},"currency":{"type":["string","null"],"pattern":"^[A-Z]{3}$"},
 "paid_date":{"type":"string","format":"date"},"external_ref":{"type":["string","null"]},
 "basis":{"enum":["receipt","user_stated","presumed_auto_charge"]},
 "match_status":{"enum":["matched_ref","matched_fallback","matched_user","presumed","unmatched","ambiguous","prior_cycle"]},
 "matched_obligation_id":{"type":["string","null"]},"cycle_due_date":{"type":["string","null"],"format":"date"},"logged_at":{"type":"string","format":"date-time"}}}
```

```json
{"$id":"schema://nova/auto-mark-bills-paid/reconcile-result@0.1.0","type":"object","required":["payment_id","outcome","match_status"],
 "properties":{"payment_id":{"type":["string","null"]},
 "outcome":{"enum":["closed_one_off","advanced_cycle","logged_unmatched","logged_ambiguous","payment_dup","skipped_receive","presumed_closed","blocked"]},
 "match_status":{"type":"string"},"obligation_id":{"type":["string","null"]},
 "previous_due_date":{"type":["string","null"],"format":"date"},"new_due_date":{"type":["string","null"],"format":"date"},
 "candidates":{"type":"array","items":{"type":"string"}},"limitations":{"type":"array","items":{"type":"string"}}}}
```

Supporting schemas (one per line):

```json
{"$id":"schema://nova/auto-mark-bills-paid/obligation-settle-view@0.1.0","type":"object","required":["obligation_id","series_key","due_date","recurrence","auto_charge","status"],"properties":{"obligation_id":{"type":"string"},"series_key":{"type":"string"},"external_ref":{"type":["string","null"]},"amount":{"type":["string","null"],"pattern":"^[0-9]{1,13}(\\.[0-9]{1,4})?$"},"currency":{"type":["string","null"],"pattern":"^[A-Z]{3}$"},"due_date":{"type":"string","format":"date"},"recurrence":{"enum":["weekly","monthly","quarterly","annual","one_off","unknown"]},"auto_charge":{"type":"boolean"},"status":{"enum":["upcoming","due_today","overdue","paid","dismissed","snoozed","needs_date","needs_review"]},"paid_basis":{"enum":["receipt","user_stated","presumed_auto_charge",null]},"last_paid_at":{"type":["string","null"],"format":"date"}}}
{"$id":"schema://nova/auto-mark-bills-paid/obligation-query@0.1.0","type":"object","properties":{"obligation_ids":{"type":"array","items":{"type":"string"}},"statuses":{"type":"array","items":{"type":"string"}},"limit":{"type":"integer","minimum":1,"maximum":500}}}
{"$id":"schema://nova/auto-mark-bills-paid/source-ref@0.1.0","type":"object","required":["source_ref"],"properties":{"source_ref":{"type":"string","pattern":"^(msg|user|auto):[A-Za-z0-9._-]{4,80}$"}}}
{"$id":"schema://nova/auto-mark-bills-paid/match-decision@0.1.0","type":"object","required":["decision","candidates"],"properties":{"decision":{"enum":["settle","ambiguous","unmatched","prior_cycle"]},"variant":{"enum":["user-stated","ref","series-amount-timing",null]},"candidates":{"type":"array","items":{"type":"object","required":["obligation_id","due_date"],"properties":{"obligation_id":{"type":"string"},"due_date":{"type":"string","format":"date"},"amount_delta_pct":{"type":["string","null"]},"day_offset":{"type":["integer","null"]}}}}}}
{"$id":"schema://nova/auto-mark-bills-paid/commit-input@0.1.0","type":"object","required":["event","decision"],"properties":{"event":{"$ref":"schema://nova/auto-mark-bills-paid/payment-event@0.1.0"},"decision":{"enum":["settle","ambiguous","unmatched","prior_cycle"]},"obligation_id":{"type":["string","null"]},"expected_due_date":{"type":["string","null"],"format":"date"}}}
{"$id":"schema://nova/auto-mark-bills-paid/link-input@0.1.0","type":"object","required":["payment_id","approval_id"],"properties":{"payment_id":{"type":"string"},"obligation_id":{"type":["string","null"],"description":"null = the user chose none"},"approval_id":{"type":"string"}}}
{"$id":"schema://nova/auto-mark-bills-paid/sweep-input@0.1.0","type":"object","required":["today","grace_days"],"properties":{"today":{"type":"string","format":"date"},"grace_days":{"type":"integer","minimum":0,"maximum":14},"enabled":{"type":"boolean"}}}
{"$id":"schema://nova/auto-mark-bills-paid/sweep-result@0.1.0","type":"object","required":["closed"],"properties":{"closed":{"type":"array","items":{"type":"object","required":["obligation_id","payment_id"],"properties":{"obligation_id":{"type":"string"},"payment_id":{"type":"string"}}}},"skipped_failed_payment":{"type":"array","items":{"type":"string"}}}}
{"$id":"schema://nova/auto-mark-bills-paid/delivery@0.1.0","type":"object","required":["delivered"],"properties":{"delivered":{"type":"boolean"},"message_id":{"type":["string","null"]},"error":{"type":["string","null"]}}}
```

**Example (fictional)**

Input: `{"source_ref":"msg:ex7c21","basis":"receipt","direction":"pay","merchant":"Example Cloud Suite","series_key":"example-cloud-suite","amount":"9.99","currency":"USD","paid_date":"2026-11-07","external_ref":"INV-2026-1107","confidence":"high"}`

Output: `{"payment_id":"pay_0042","outcome":"advanced_cycle","match_status":"matched_ref","obligation_id":"obl_0011","previous_due_date":"2026-11-07","new_due_date":"2026-12-07","candidates":["obl_0011"],"limitations":[]}`

## E. Constraints and budgets

- **Hard constraints:** never create, change or delete finance-hub records or balances; never pay or move money; `direction = receive` is never treated as a payment; amounts are decimal strings with an explicit currency; a fallback match needs exactly one qualifying candidate; a receipt never moves a cycle backwards and never closes a cycle whose window it is outside; presumed closures are always labelled presumed; one log row per `source_ref`.
- **Soft constraints:** prefer reference matches; keep the log readable ("Example Cloud Suite, 9.99 USD, 7 Nov").
- **Resource and tenant scope:** the requesting user's obligations and payments log only.
- **Data sources allowed:** the payment event, the obligation store, the payments log.
- **External transfers permitted:** none, except the optional ambiguity notice to the user.
- **Time limit:** 30 seconds per event; 2 minutes per sweep.
- **Cost or token budget:** no LLM in this recipe (extraction happens upstream).
- **Maximum steps:** 9 (S1–S9).
- **Maximum retries and correction cycles:** commit 2 retries after a read-back; reads 2 retries; correction cycles: 1.
- **Rate limits and concurrency:** commits serialised per obligation (row lock); sweep runs at most once per reminder run.
- **Retention and logging constraints:** the payments log is kept as a financial history per operator policy (`TBD`); logs never hold message bodies.
- **Prohibited actions:** marking paid on a fuzzy series match without amount and timing agreement; overwriting `paid_basis = user_stated` with an inferred basis; deleting log rows.
- **Stop conditions:** invalid event; store unavailable; kill switch.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Payments-log row | valid event; `direction = pay`; `source_ref` not yet logged | Allowed (standing scope) | read-back by `source_ref` |
| Read | Obligations, payments log | own tenant | Allowed | — |
| Update | Obligation one-off: `status = paid`, `paid_at`, `paid_basis` | unique ref, fallback or user-stated match | Allowed | read-back |
| Update | Obligation recurring: `due_date` + one period, cycle flags reset, `last_paid_at` | unique match; `paid_date` within the window | Allowed | new due = old due + one period |
| Update | Obligation one-off auto-charge: `status = paid`, `paid_basis = presumed_auto_charge` | sweep rules; no failed-payment notice | Allowed (presumed-paid enabled) | read-back + `auto:` log row |
| Update | Log row link to an obligation (`link`) | ambiguous or unmatched row | Approval Required (the user picks) | read-back |
| Delete | any log row or obligation | — | Not Allowed | — |
| External side effect | ambiguity notice to the user | ambiguous outcome | Allowed | delivery receipt |

- **Idempotency:** `source_ref` is unique in the payments log; a re-delivered message returns `payment_dup` and never advances a cycle twice. The commit's obligation update is conditional on the cycle's `due_date` still matching the value read in S3/S4.
- **Expected side effects:** a log row; one obligation closed or advanced; at most one notice.
- **Compensation:** the user can reopen an obligation ("not paid"). This restores the previous status and due date from history and sets the log row's `match_status` to `unmatched` with an audit note. The log row is kept.
- **Irreversible:** none within this recipe (everything is a reversible record change); a delivered notice.
- **Ambiguous outcomes:** if `commit` times out, read by `source_ref`; if the row exists, verify the obligation state and do not retry.

## G. Autonomy and approval policy

- **Default autonomy level:** A2 — bounded autonomy on reminder-state records only.
- **Planning level:** A2.
- **Execution level:** A2 for unique matches and presumed closures; A0 for ambiguous matches.
- **Verification level:** A2.
- **Recovery level:** A2.
- **Approval required for:** linking an ambiguous or unmatched payment to an obligation; turning presumed-paid on or off; reopening.
- **Actions never permitted:** changing balances or finance-hub records; paying; treating incoming money as a bill payment; choosing between candidates.
- **Conditions requiring clarification:** more than one candidate; a user-stated payment for an obligation that is already paid.
- **Conditions requiring escalation:** a failed-payment notice arrives for an obligation already marked presumed paid (reopen it and tell the user).
- **Safe state after stop, cancellation or escalation:** either the log row and obligation update are both committed, or neither is; ambiguous rows are logged without touching obligations.

## H. Execution plan

**S1 — Validate the payment event**
1. Objective: accept only well-formed outgoing payments.
2. Capability: none (local validation).
3. Input mapping: hand-off or user message → `payment-event@0.1.0`.
4. Dependencies: none.
5. Preconditions: kill switch off.
6. Action / side effects: validate; `direction = receive` → outcome `skipped_receive` (stop).
7. Autonomy: A2.
8. Timeout / retry: 2 s / none.
9. Idempotency: not applicable.
10. Postconditions: a valid `pay` event.
11. Verification: schema valid; currency present whenever amount present; `paid_date` ≤ today + 1 day.
12. Failure routes: invalid → `blocked`, nothing logged; a future `paid_date` → `blocked` ("receipt dated in the future").
13. Escalation triggers: none.
14. Evidence: event hash and validation result.

**S2 — Check for a duplicate**
1. Objective: never process the same receipt twice.
2. Capability: `nova.finance.payments.get_by_source`.
3. Input mapping: `source_ref`.
4. Dependencies: S1.
5. Preconditions: valid event.
6. Action / side effects: read-only.
7. Autonomy: A2.
8. Timeout / retry: 5 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: duplicate yes/no.
11. Verification: lookup returns 0 or 1 rows.
12. Failure routes: exists → outcome `payment_dup`, `completed` (no further steps).
13. Escalation: none.
14. Evidence: existing `payment_id` if any.

**S3 — Match by reference (or user statement)**
1. Objective: the strongest possible link.
2. Capability: `nova.finance.payments.match` (variants `user-stated`, then `ref`).
3. Input mapping: `obligation_id` (user-stated) or normalised `external_ref`.
4. Dependencies: S2.
5. Preconditions: a reference or a user-named obligation exists; otherwise skip to S4.
6. Action / side effects: read-only search among open obligations (upcoming, due_today, overdue, snoozed, needs_date).
7. Autonomy: A2.
8. Timeout / retry: 5 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: 0, 1 or more candidates.
11. Verification: for user-stated, the obligation belongs to the tenant and is open; for ref, an exact normalised equality holds.
12. Failure routes: user-stated obligation already paid → ask "it is already marked paid on …; log this payment anyway?"; 0 → S4.
13. Escalation: none.
14. Evidence: candidate IDs and match variant.

**S4 — Fallback match by series, amount and timing**
1. Objective: link receipts that carry no reference.
2. Capability: `nova.finance.payments.match.series-amount-timing`.
3. Input mapping: `series_key`, `currency`, `amount`, `paid_date`, tolerance, window.
4. Dependencies: S3 with zero candidates.
5. Preconditions: `amount` and `currency` present (otherwise outcome `logged_unmatched`).
6. Action / side effects: read-only.
7. Autonomy: A2.
8. Timeout / retry: 5 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: 0, 1 or more candidates, each with its cycle `due_date`.
11. Verification: each candidate satisfies all four conditions, recomputed independently from the stored row.
12. Failure routes: `paid_date` before the window of the only series match → `prior_cycle` (logged, no cycle change).
13. Escalation: none.
14. Evidence: candidate IDs with the computed amount delta and day offset.

**S5 — Decide**
1. Objective: one deterministic outcome.
2. Capability: decision rules (part of the planner; also enforced inside `commit`).
3. Input mapping: candidates from S3/S4.
4. Dependencies: S3, S4.
5. Preconditions: none.
6. Action: exactly 1 candidate → settle (one-off → `closed_one_off`; periodic → `advanced_cycle`); more than 1 → `logged_ambiguous`; 0 → `logged_unmatched` or `prior_cycle`.
7. Autonomy: A2.
8. Timeout / retry: not applicable.
9. Idempotency: pure function.
10. Postconditions: decision + the expected `due_date` of the cycle being settled.
11. Verification: the decision is recomputed inside the commit function; a mismatch aborts.
12. Failure routes: none.
13. Escalation: ambiguous → S9 notice.
14. Evidence: decision record.

**S6 — Commit atomically**
1. Objective: log and settle together, or not at all.
2. Capability: `nova.finance.payments.commit.default`.
3. Input mapping: event, decision, obligation ID, expected cycle `due_date`.
4. Dependencies: S5.
5. Preconditions: the obligation's `due_date` still equals the expected value (row lock).
6. Action / side effects: insert the log row; for one-off set `status = paid`, `paid_at`, `paid_basis`; for periodic, add one period to `due_date`, reset cycle flags and set `last_paid_at` (if the new date is already past, the status becomes `overdue`; nothing is skipped); write history.
7. Autonomy: A2 within the standing scope.
8. Timeout / retry: 10 s / 2 retries after reading back by `source_ref`.
9. Idempotency: unique `source_ref`; cycle guard.
10. Postconditions: log row + settled obligation, or log row only.
11. Verification: the function returns `{payment_id, outcome, new_due_date}`.
12. Failure routes: cycle guard fails (the obligation changed meanwhile) → re-run S3–S5 once; still failing → log only as `ambiguous`.
13. Escalation: none.
14. Evidence: payment ID, obligation history row ID.

**S7 — Verify**
1. Objective: prove the state change is the intended one.
2. Capability: `nova.finance.obligations.list`, `nova.finance.payments.get_by_source`.
3. Input mapping: `payment_id`, `obligation_id`.
4. Dependencies: S6.
5. Preconditions: commit returned.
6. Action: read both rows.
7. Autonomy: A2.
8. Timeout / retry: 5 s / 1 retry.
9. Idempotency: read-only.
10. Postconditions: verified or flagged.
11. Verification: the log row exists once; a one-off is `paid` with a basis; for periodic, new `due_date` = old + exactly one period and the flags are reset; no finance-hub record was created (scope check).
12. Failure routes: mismatch → restore from history, mark the log row `ambiguous`, outcome `partially_completed`.
13. Escalation: repeated mismatches → disable auto-settling (log-only mode) and tell the user.
14. Evidence: read-back snapshot.

**S8 — Presumed-paid sweep (scheduled branch)**
1. Objective: stop auto-charged one-off bills from showing as overdue.
2. Capability: `nova.finance.obligations.presumed_paid_sweep.default`.
3. Input mapping: today (local), `grace_days`.
4. Dependencies: runs with the reminder schedule; independent of S1–S7.
5. Preconditions: presumed-paid enabled; candidates are `auto_charge = true`, recurrence `one_off` or `unknown`, `due_date` + `grace_days` < today, not paid or dismissed, and no failed-payment notice for the series since the due date.
6. Action / side effects: close each as `paid` with `paid_basis = presumed_auto_charge`; insert log rows `auto:<obligation_id>`.
7. Autonomy: A2.
8. Timeout / retry: 30 s / 2 retries.
9. Idempotency: the `auto:<obligation_id>` source reference is unique.
10. Postconditions: no eligible row remains open.
11. Verification: count of eligible rows after the sweep = 0; every closed row has a matching `auto:` log row.
12. Failure routes: error → leave rows open (the reminder recipe never sends overdue alerts for auto-charge rows).
13. Escalation: a failed-payment notice arrives later → reopen, tell the user (Section L).
14. Evidence: closed IDs.

**S9 — Report**
1. Objective: an honest result and, if needed, one question to the user.
2. Capability: `nova.notify.owner.send` (ambiguous only).
3. Input mapping: `reconcile_result`.
4. Dependencies: S7 (or S1/S2 on the short paths).
5. Preconditions: none.
6. Action / side effects: store the result; for `logged_ambiguous`, send "A 9.99 USD payment to Example Cloud Suite on 7 Nov could settle 2 bills — which one?" with the candidates.
7. Autonomy: A2; resolving is A0.
8. Timeout / retry: 15 s / 2 retries.
9. Idempotency: notice key = `payment_id`.
10. Postconditions: result stored.
11. Verification: the result matches the read-back.
12. Failure routes: notice fails → the row stays `ambiguous` and visible in the schedule view; `partially_completed`.
13. Escalation: none.
14. Evidence: result record, notice message ID.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Input validity | event matches `payment-event@0.1.0`; outgoing only | validator | valid | `blocked` / `skipped_receive` |
| Idempotency | one log row per `source_ref` | unique constraint + S2 | holds | `payment_dup` |
| Match strength | auto-settle only on a user statement, an exact ref, or a unique four-condition fallback | S3/S4 records | holds | log only |
| Timing gate | `paid_date` within [due − 5, due + 45] for fallback matches | S4 offsets | holds | `prior_cycle` |
| Cycle arithmetic | new due = old due + one period | S7 read-back | exact | restore, flag |
| No balance effect | zero finance-hub writes in the run | scope audit log | zero | stop, report defect |
| Presumed label | every sweep closure has `paid_basis = presumed_auto_charge` | S8 read-back | 100 % | fix label |
| Output schema | `reconcile_result` valid | validator | valid | fix; never fake outcomes |

A successful commit proves the rows changed. It does not prove the bill was really paid; the payment evidence comes from the source message, and presumed closures carry their own label.

## J. Self-correction and recovery

- **Failure classes:** validation, duplicate, store unavailable, cycle-guard conflict, failed read-back, failed-payment contradiction.
- **Permitted corrections:** re-run matching once after a cycle-guard conflict; restore from history after a failed verification; reopen a presumed closure when a failed-payment notice arrives.
- **Forbidden corrections:** widening the amount tolerance or timing window to force a match; picking one of several candidates; relabelling a presumed closure as confirmed; touching finance-hub records.
- **Maximum correction cycles:** 1 per event.
- **Reverification required after correction:** yes — S7.
- **Stop conditions:** correction budget exhausted; kill switch.
- **Recovery or compensation:** history-based restore; the user can reopen any obligation.
- **What must be reported:** ambiguous and unmatched payments, restores, reopened presumed closures.

## K. Independent challenge

- **Assumptions to challenge:** the receipt is for the current cycle and not the previous one; the reference number belongs to this biller; a "renewed" message means money actually left the account.
- **Alternative explanation:** a "payment received" email may be the user's own incoming revenue (a payment-gateway notice to a merchant), not a bill they paid. This is why `receive` is excluded.
- **Contradictory evidence:** a later failed-payment or "payment declined" notice; a refund for the same reference.
- **Could mislead:** "paid" shown for a presumed closure. The view must say "presumed paid (auto-charge)".
- **Separate verifier:** the commit function recomputes the decision from stored rows, independently of S3–S5; S7 re-reads both rows.
- **Findings requiring disclosure:** presumed closures, prior-cycle receipts, ambiguous matches.

## L. Human escalation

- **Triggers:** more than one candidate; a user-stated payment for an already-paid bill; a failed-payment notice after a presumed closure; repeated verification failures.
- **Why the agent cannot decide:** choosing the wrong bill hides a real debt.
- **Information presented:** payment (merchant, amount + currency, date, reference) and candidate obligations (due date, amount).
- **Decision requested:** which obligation this settles, or "none".
- **Options and consequences:** linking closes or advances that obligation; "none" keeps the payment as an unmatched historical record.
- **Safe behaviour while waiting:** the payment is logged; no obligation changes; reminders continue for all candidates.
- **If no response:** the row stays `ambiguous`; nothing auto-resolves.
- **Resume:** `nova.finance.payments.link` with the user's choice → S6 commit and S7 verification.

## M. Completion conditions

Mandatory: `reconcile_result` valid; at most one log row per `source_ref`; any settlement verified by read-back; no finance-hub write; presumed closures labelled; ambiguity disclosed; evidence references present.

- `completed` — the event was logged and either settled with verification, logged as unmatched/prior-cycle, or recognised as a duplicate; or the sweep closed all eligible rows.
- `partially_completed` — logged but the settlement failed verification and was restored, or the ambiguity notice could not be delivered.
- `blocked` — invalid event or missing time zone.
- `failed` — the store was unavailable; nothing committed.
- `cancelled` — kill switch, or the user declined to link.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome ("Example Cloud Suite paid; next due 7 Dec"); `reconcile_result`; step statuses S1–S9; verification results and failed checks; evidence references (payment ID, obligation history row); approvals and scope (standing scope ID, link approvals); side effects (rows changed) and reconciliation status; corrections performed; unresolved challenges (ambiguous candidates); limitations; escalation state; learning candidate. Never include message bodies or full account numbers.

## O. Learning and revision

- **Outcome metrics:** auto-settle rate by variant (user-stated / ref / fallback), ambiguous rate, user reopen rate (wrong settlement), prior-cycle count, presumed closures later contradicted.
- **Evidence retained:** payments log and obligation history, per the retention policy.
- **Data minimisation:** no message text; amounts only in the log, which is the user's own financial record.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}` — for example "billers that put the order number in the subject only → extract the reference from the subject".
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests; a tolerance or window change also needs the prior-cycle fixtures.
- **Promotion approval:** publisher review plus Skill Port review.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path — reference match | obligation closed or advanced; log row created | UNVERIFIED on Skill Port; observed in the reference implementation (rolled-back test) |
| Happy path — fallback match | unique series + amount + timing → advanced | UNVERIFIED on Skill Port; observed in the reference implementation |
| Invalid input (amount without currency) | `blocked`; nothing logged | TBD |
| Missing input (no `source_ref`) | `blocked` | TBD |
| Incoming revenue notice | `skipped_receive` | TBD |
| Missing capability (commit binding absent) | `blocked`; nothing logged | TBD |
| Permission denied (`finance:obligations:settle` missing) | log-only mode or `blocked` per config; no settle | TBD |
| Unauthorized scope (another tenant's obligation ID, user-stated) | rejected | TBD |
| Tenant isolation | matching never sees other tenants' rows | TBD |
| Prior-cycle receipt | `prior_cycle`; cycle unchanged | TBD |
| Two candidates | `logged_ambiguous`; user asked; nothing settled | TBD |
| Timeout on commit | read-back by `source_ref`; no double advance | TBD |
| Duplicate / replay (same message twice) | `payment_dup`; one advance only | TBD |
| False success (commit OK, read-back shows no change) | restored, flagged, `partially_completed` | TBD |
| Failed verification (wrong period arithmetic) | restored | TBD |
| Correction budget exhausted | log only; stop | TBD |
| Presumed-paid sweep | one-off auto-charge past grace → presumed paid, labelled | TBD |
| Failed-payment notice after presumed closure | reopened; user told | TBD |
| Escalation (ambiguous) | notice sent; waits | TBD |
| Cancellation (user answers "none") | stays unmatched | TBD |
| Compensation (user reopens) | previous state restored; log row kept | TBD |
| No balance effect | zero finance-hub writes across all tests | TBD |
| Output schema and status | valid result; correct status | TBD |
| Compatibility / regression | fixtures pass after store migration | TBD |

All tests use fictional obligations and receipts in rolled-back transactions; never real bills or real accounts. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, schema registry and ambiguity UI — TBD (owner: publisher, with Skill Port).
- Default amount tolerance (12 %) suits currency conversion and tax differences, but could link the wrong bill when the same biller has two similar plans; review after collecting metrics (owner: publisher).
- Renewal messages without an amount or reference can only match by a user statement or presumed-paid logic.
- Partial payments (for example a card minimum payment) are treated as payments of the full amount if they fall within tolerance. Card minimum payments are outside tolerance, so they log as unmatched. Proper partial-payment support is TBD.
- The failed-payment check depends on the radar classifying failed-payment notices; this classification is TBD in the extractor.
