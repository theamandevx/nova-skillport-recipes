# Subscription audit

## A. Identity and ownership

- **Recipe ID:** `nova.finance.subscription-audit` (proposed until assigned by Skill Port)
- **Name:** Subscription audit
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** finance
- **Tags:** subscriptions, recurring-charges, audit, monthly-digest, duplicates, price-increase, read-only · Vedaxi task type: none (no supported task type covers this today)
- **Summary:** Once a month, the agent scans the user's transaction history, payments log and payment schedule for recurring charges. It works out each subscription's cadence and monthly-equivalent cost, totals them per currency and in one reporting currency, and flags duplicates, price increases, trials that turned paid, and charges after a cancellation. It sends a short digest with suggestions. It is strictly read-only: it never cancels anything and never changes a record.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** Read access to finance-hub records, the payments log (`nova.finance.auto-mark-bills-paid`) and the obligation schedule (`nova.finance.upcoming-payments-radar`); exchange rates from `nova.finance.daily-exchange-rates`; an LLM for wording only; the owner channel. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — a monthly digest built from the schedule and payments history ran in Nova's reference implementation and produced findings (catalogue status: Built). Not every control described here has evidence yet (Section P lists the tests still needed). Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.finance.hub` | Capability | Finance-hub records | medium | UNVERIFIED |
| `nova.finance.hub.records_query` | Operation (read) | Expense records in a date window, per account | medium | UNVERIFIED |
| `nova.finance.payments` | Capability | Payments log | low | UNVERIFIED |
| `nova.finance.payments.list` | Operation (read) | Payments in a date window | low | UNVERIFIED |
| `nova.finance.obligations` | Capability | Payment schedule | low | UNVERIFIED |
| `nova.finance.obligations.list` | Operation (read) | Recurring obligations, including dismissed and cancelled series | low | UNVERIFIED |
| `nova.finance.subscriptions` | Capability | Deterministic recurring-charge analysis | low | UNVERIFIED |
| `nova.finance.subscriptions.analyse` | Operation (read, compute) | Build series, cadences, monthly equivalents, totals and findings | low | UNVERIFIED |
| `nova.finance.subscriptions.save_report` | Operation (create) | Store the audit report (its only write) | low | UNVERIFIED |
| `nova.finance.fx` | Capability | Exchange-rate snapshots | low | UNVERIFIED |
| `nova.finance.fx.get_rates` | Operation (read) | Latest verified snapshot, with staleness status | low | UNVERIFIED |
| `nova.finance.advise` | Capability | LLM wording of suggestions | low | UNVERIFIED |
| `nova.finance.advise.word_suggestions` | Operation (read, LLM) | Turn findings into short suggestions; numbers locked | low | UNVERIFIED |
| `nova.notify.owner` | Capability | Owner channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Deliver the digest | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.finance.subscriptions.analyse.default` — deterministic rules (Section H, S3–S6) with decimal arithmetic. Only the default variant is supported.
- `nova.finance.advise.word_suggestions.llm` — an LLM writes one or two sentences per finding from the finding's JSON. Every number in the output must appear in the input.
- `nova.finance.advise.word_suggestions.template` — fixed templates, used when the LLM is unavailable or its output fails verification. Both variants are read-only and produce the same schema, so the fallback is safe.
- `nova.finance.fx.get_rates.default` — read-only; returns `fresh | stale | expired | unverified`.

**Bindings:** `binding://nova/finance/records-query`, `binding://nova/finance/payments-list`, `binding://nova/finance/obligations-list`, `binding://nova/finance/subscriptions-analyse`, `binding://nova/finance/subscriptions-report`, `binding://nova/finance/fx-get`, `binding://nova/finance/advise-suggestions`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: a monthly automation-hub job (1st of the month, 08:00 local) reads a recurring-charge registry function, asks an LLM to flag duplicates/overlaps, price increases and trial conversions, and sends a chat digest.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Hub record | `hub-record@0.1.0` (email-logger schema) | the requesting user's tenant | read |
| Payments-log row | `payment-log@0.1.0` (paid-tracking schema) | the requesting user's tenant | read |
| Obligation | `obligation@0.1.0` (radar schema) | the requesting user's tenant | read |
| FX snapshot | `fx-snapshot@0.1.0` (exchange-rates schema) | tenant or shared reference data | read |
| Audit report | `schema://nova/subscription-audit/audit-report@0.1.0` | the requesting user | create |

**Authorization requirements (declared, never a grant):** `finance:hub:records:read`, `finance:payments:read`, `finance:obligations:read`, `finance:fx:read`, `finance:reports:write`, `llm:advise`, `notify:owner`. Secrets: `secret_ref://llm`, `secret_ref://finance-db-role`.

**Policy / trust:** risk class `low` (read-only over financial data; owner-only output). Merchant names are untrusted text and are escaped.

## C. Intent and applicability

- **User problem:** Small recurring charges pile up, renew silently, rise in price or get duplicated across cards, and nobody adds them up.
- **Desired outcome:** A monthly picture of every recurring charge, what they cost per month in total, and a few concrete things worth checking.
- **Trigger or entry conditions:** Scheduled monthly run (default: day 1, 08:00 local), or "audit my subscriptions" on demand.
- **Applicable situations:** Streaming, software, cloud storage, memberships, telecom plans, insurance premiums, any charge that repeats on a regular cadence.
- **Out-of-scope situations:** Cancelling subscriptions; contacting merchants; changing records or the schedule; judging whether the user "uses" a service (no usage data); one-off purchases; income.
- **Required inputs:** `timezone`, `reporting_currency` (see Section D).
- **Prerequisites:** At least 3 months of history in the hub or payments log for reliable cadence detection; FX snapshots available when more than one currency is involved.
- **Required capabilities:** see Section B (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port; every binding is a dependency to resolve before activation.
- **Assumptions:** Verified in the reference implementation: a registry built from the schedule and payments history, and a monthly digest that flags duplicates, price increases and trial conversions. Not every control described here has evidence yet (Section P lists the tests still needed).
- **Ambiguity policy:** Use a documented safe default. Series that do not meet the cadence rules are listed as "possible recurring" without a monthly equivalent and are never added to the total.
- **Vedaxi task type:** none.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `timezone` | IANA zone | yes | profile | — | `blocked`. |
| `reporting_currency` | ISO 4217 | yes | profile | — | `blocked`. |
| `lookback_months` | integer 3–24 | no | user | 13 | Clamp. |
| `price_increase_pct` | number 1–50 | no | user | 5 | Clamp. |
| `amount_stability_pct` | number 5–30 | no | operator | 15 | Clamp. |
| `accounts` | array of account refs | no | user | all | Unknown refs ignored and reported. |
| `owner_channel` | channel ref | yes | installation | — | Report stored only; `partially_completed`. |

```json
{"$id":"schema://nova/subscription-audit/config@0.1.0","type":"object","required":["timezone","reporting_currency"],
 "properties":{"timezone":{"type":"string"},"reporting_currency":{"type":"string","pattern":"^[A-Z]{3}$"},
 "lookback_months":{"type":"integer","minimum":3,"maximum":24},"price_increase_pct":{"type":"number","minimum":1,"maximum":50},
 "amount_stability_pct":{"type":"number","minimum":5,"maximum":30},"accounts":{"type":"array","items":{"type":"string"}},"owner_channel":{"type":"string"}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `audit_report` | `schema://nova/subscription-audit/audit-report@0.1.0` | `report_id`, `period`, `series`, `totals`, `findings` | report store; digest to the owner channel | financial; owner only | totals recomputed independently; text numbers checked (Section I) |

```json
{"$id":"schema://nova/subscription-audit/audit-report@0.1.0","type":"object","required":["report_id","period","series","totals","findings"],
 "properties":{"report_id":{"type":"string"},"period":{"type":"object","required":["from","to"],"properties":{"from":{"type":"string","format":"date"},"to":{"type":"string","format":"date"}}},
 "series":{"type":"array","items":{"type":"object","required":["series_id","merchant","currency","cadence","last_amount","monthly_equivalent","status","charge_count"],"properties":{
  "series_id":{"type":"string"},"merchant":{"type":"string","maxLength":120},"account_ref":{"type":["string","null"]},"currency":{"type":"string","pattern":"^[A-Z]{3}$"},
  "cadence":{"enum":["weekly","monthly","quarterly","annual","possible"]},"last_amount":{"type":"string","pattern":"^[0-9]{1,13}(\\.[0-9]{1,4})?$"},
  "monthly_equivalent":{"type":["string","null"],"pattern":"^[0-9]{1,13}(\\.[0-9]{1,4})?$"},"last_charge":{"type":"string","format":"date"},
  "status":{"enum":["active","lapsed","cancelled"]},"charge_count":{"type":"integer","minimum":1},"source_refs":{"type":"array","items":{"type":"string"}}}}},
 "totals":{"type":"object","required":["by_currency"],"properties":{"by_currency":{"type":"object","additionalProperties":{"type":"string","pattern":"^[0-9]{1,13}(\\.[0-9]{1,4})?$"}},
  "converted":{"type":["object","null"],"properties":{"currency":{"type":"string","pattern":"^[A-Z]{3}$"},"amount":{"type":"string"},"fx_as_of":{"type":"string","format":"date-time"},"fx_status":{"enum":["fresh","stale","expired","unverified"]},"fx_source":{"type":"string"}}}}},
 "findings":{"type":"array","items":{"type":"object","required":["finding_id","type","series_ids"],"properties":{"finding_id":{"type":"string"},
  "type":{"enum":["duplicate","possible_overlap","price_increase","trial_converted","charged_after_cancellation","lapsed"]},"series_ids":{"type":"array","items":{"type":"string"}},
  "old_amount":{"type":["string","null"]},"new_amount":{"type":["string","null"]},"change_pct":{"type":["string","null"]},"suggestion":{"type":"string","maxLength":300},"suggestion_source":{"enum":["llm","template"]}}}},
 "limitations":{"type":"array","items":{"type":"string"}}}}
```

**Rules:** cadence by intervals between charges — weekly 6–8 days (≥ 3 intervals), monthly 26–35 days (≥ 2 intervals, or 1 interval plus a schedule entry saying monthly), quarterly 85–95 days (≥ 1 interval plus a schedule entry, or ≥ 2 intervals), annual 350–380 days (1 interval) or a schedule entry saying annual. Every amount must be within ± `amount_stability_pct` of the series median, except a price step that is reported as a finding. Monthly equivalent = weekly × 52 ÷ 12, monthly × 1, quarterly ÷ 3, annual ÷ 12. Decimal arithmetic is used throughout, rounded half-even to 2 places only at output.

Reused schemas (acknowledged): `hub-record@0.1.0` from `nova.finance.log-transactions-from-email` (folder `15-log-transactions-from-email`); `payment-log@0.1.0` from `nova.finance.auto-mark-bills-paid` (`14-auto-mark-bills-paid`); `obligation@0.1.0` from `nova.finance.upcoming-payments-radar` (`12-upcoming-payments-radar`); `fx-query@0.1.0` and `fx-snapshot@0.1.0` from `nova.finance.daily-exchange-rates` (`18-daily-exchange-rates`). Own supporting schemas:

```json
{"$id":"schema://nova/subscription-audit/records-query@0.1.0","type":"object","required":["from","to"],"properties":{"from":{"type":"string","format":"date"},"to":{"type":"string","format":"date"},"accounts":{"type":"array","items":{"type":"string"}},"direction":{"const":"expense"},"limit":{"type":"integer","minimum":1,"maximum":20000}}}
{"$id":"schema://nova/subscription-audit/delivery@0.1.0","type":"object","required":["delivered"],"properties":{"delivered":{"type":"boolean"},"message_id":{"type":["string","null"]},"report_id":{"type":"string"},"error":{"type":["string","null"]}}}
```

**Example (fictional, run 2026-12-01, reporting currency USD)**

Series: Example Streaming 12.99 USD monthly (card ··0142); Example Music 9.99 USD monthly on card ··0142 **and** 9.99 USD monthly on bank ··7781; Example Cloud Suite 99.00 USD annual (8.25/month); Example News 4.00 EUR monthly.

Output (abridged): `{"report_id":"aud_2026_12","period":{"from":"2025-11-01","to":"2026-11-30"},"totals":{"by_currency":{"USD":"41.22","EUR":"4.00"},"converted":{"currency":"USD","amount":"45.57","fx_as_of":"2026-11-30T16:00:00Z","fx_status":"fresh","fx_source":"primary rates provider (mid-market)"}},"findings":[{"finding_id":"f1","type":"duplicate","series_ids":["s2","s3"],"suggestion":"Example Music is billed on two accounts (9.99 USD each). If one is a family or old plan, consider keeping just one.","suggestion_source":"llm"},{"finding_id":"f2","type":"price_increase","series_ids":["s1"],"old_amount":"10.99","new_amount":"12.99","change_pct":"18.2","suggestion":"Example Streaming went from 10.99 to 12.99 USD (+18.2%). Worth checking whether a cheaper plan fits.","suggestion_source":"llm"}],"limitations":[]}`

## E. Constraints and budgets

- **Hard constraints:** read-only over records, payments, obligations and accounts (the only write is the report); never cancel, unsubscribe, contact merchants or move money; amounts are decimal strings with an explicit currency; per-currency totals are always shown, and a converted total is shown only with its FX as-of time, source and status (`expired` FX → no converted total); "possible" series are never added to totals; suggestions contain no number that is not in the finding and make no usage claims ("you don't use this").
- **Soft constraints:** at most 5 findings in the digest (all in the report); friendly, short wording.
- **Resource and tenant scope:** the requesting user's records, payments, obligations and reports.
- **Data sources allowed:** finance hub, payments log, obligation schedule, FX snapshots.
- **External transfers permitted:** finding JSON (merchant names and amounts, no account numbers) to the approved LLM; the digest to the owner channel.
- **Time limit:** 5 minutes.
- **Cost or token budget:** one LLM call per run (all findings in one prompt), token cap `TBD`.
- **Maximum steps:** 10 (S1–S10).
- **Maximum retries and correction cycles:** reads 2; LLM 1 retry, then the template; send 2; correction cycles 1.
- **Rate limits and concurrency:** one run per user per month (on-demand runs are allowed but cannot overwrite the monthly report).
- **Retention and logging constraints:** reports kept per policy (`TBD`); logs hold counts only.
- **Prohibited actions:** any write to financial data; claiming savings that are not computed from the data.
- **Stop conditions:** no history in the window (`blocked` with an explanation); kill switch.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Audit report (one per period; on-demand reports are versioned) | analysis verified | Allowed | read-back |
| Read | Records, payments, obligations, FX snapshots | own tenant; lookback window | Allowed | — |
| Update | any financial object | — | Not Allowed | — |
| Delete | any object | — | Not Allowed | — |
| External side effect | digest to the owner channel | report stored | Allowed | receipt |
| External side effect | finding JSON to the LLM | approved provider | Allowed | call log |

- **Idempotency:** report key = `(user, period)`; the scheduled run re-uses the key; on-demand runs create `(user, period, vN)`. Digest key = report ID.
- **Expected side effects:** one report; one message.
- **Compensation:** not applicable — no financial state changes; a wrong report can be superseded by a new version.
- **Irreversible:** the delivered digest; text sent to the LLM.
- **Ambiguous outcomes:** if the send times out, do not resend in the same run; the report is stored and visible.

## G. Autonomy and approval policy

- **Default autonomy level:** A2 — bounded autonomy (read-only analysis, owner-only output).
- **Planning level:** A2.
- **Execution level:** A2.
- **Verification level:** A2.
- **Recovery level:** A2.
- **Approval required for:** enabling the monthly schedule (one-time); nothing else, because nothing is changed.
- **Actions never permitted:** cancelling, unsubscribing, contacting merchants, editing or deleting records, paying.
- **Conditions requiring clarification:** none at run time.
- **Conditions requiring escalation:** none — findings are informational. A charge after a cancellation is highlighted at the top of the digest.
- **Safe state after stop, cancellation or escalation:** no report or a complete report; never a partial one marked complete.

## H. Execution plan

**S1 — Validate and set the window**
1. Objective: a valid configuration and a clear analysis period.
2. Capability: none (local validation).
3. Input mapping: config → `config@0.1.0`; period = the last `lookback_months` full months.
4. Dependencies: none.
5. Preconditions: kill switch off.
6. Action / side effects: validate; take lock `subs-audit:<user>`.
7. Autonomy: A2.
8. Timeout / retry: 5 s / none.
9. Idempotency: report key `(user, period)`.
10. Postconditions: period `{from, to}`.
11. Verification: schema valid.
12. Failure routes: invalid → `blocked`.
13. Escalation triggers: none.
14. Evidence: config hash, period.

**S2 — Read history**
1. Objective: every charge that might recur.
2. Capability: `nova.finance.hub.records_query`, `nova.finance.payments.list`, `nova.finance.obligations.list`.
3. Input mapping: period, `accounts`.
4. Dependencies: S1.
5. Preconditions: read scopes.
6. Action / side effects: read-only; expenses only; obligations including dismissed or cancelled series.
7. Autonomy: A2.
8. Timeout / retry: 30 s each / 2.
9. Idempotency: read-only.
10. Postconditions: charge list with source refs; obligation list.
11. Verification: all rows belong to the tenant; dates within the period.
12. Failure routes: one source down → continue with the others and disclose; all down → `failed`; empty → `blocked` ("not enough history").
13. Escalation: none.
14. Evidence: row counts per source.

**S3 — Normalise and group**
1. Objective: one group per merchant, currency and account.
2. Capability: `nova.finance.subscriptions.analyse` (normalisation stage).
3. Input mapping: charges.
4. Dependencies: S2.
5. Preconditions: none.
6. Action: lower-case; strip punctuation, processor prefixes and trailing reference digits; use `series_key` where known; drop payments-log rows that duplicate a hub record (same date ± 3 days, same amount); group.
7. Autonomy: A2.
8. Timeout / retry: 10 s / none.
9. Idempotency: pure.
10. Postconditions: groups with sorted charges.
11. Verification: every charge is in exactly one group.
12. Failure routes: none.
13. Escalation: none.
14. Evidence: group count, de-duplicated count.

**S4 — Detect recurring series**
1. Objective: honest cadences.
2. Capability: `nova.finance.subscriptions.analyse` (cadence stage).
3. Input mapping: groups, `amount_stability_pct`, schedule recurrence hints.
4. Dependencies: S3.
5. Preconditions: none.
6. Action: apply the cadence rules (Section D); mark status `active` (last charge within one cadence + 7 days), `lapsed` (≥ 2 missed cycles) or `cancelled` (series dismissed after a cancellation); groups that fail the rules but have ≥ 2 charges → `possible`.
7. Autonomy: A2.
8. Timeout / retry: 10 s / none.
9. Idempotency: pure.
10. Postconditions: series list.
11. Verification: every non-possible series meets its rule (rechecked in S8).
12. Failure routes: none.
13. Escalation: none.
14. Evidence: series with intervals.

**S5 — Compute totals**
1. Objective: correct monthly-equivalent totals.
2. Capability: `nova.finance.subscriptions.analyse` (totals stage); `nova.finance.fx.get_rates`.
3. Input mapping: active series, `reporting_currency`.
4. Dependencies: S4.
5. Preconditions: none.
6. Action: monthly equivalent per active series; sum per currency; if more than one currency, read the latest FX snapshot — `fresh` or `stale` → converted total with as-of time and status; `expired` or `unverified` → per-currency totals only.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 for FX.
9. Idempotency: pure plus a read.
10. Postconditions: `totals`.
11. Verification: recomputed independently in S8.
12. Failure routes: FX unavailable → per-currency only, disclosed.
13. Escalation: none.
14. Evidence: FX snapshot ID.

**S6 — Detect findings**
1. Objective: the few things worth the user's attention.
2. Capability: `nova.finance.subscriptions.analyse` (findings stage).
3. Input mapping: series, obligations, `price_increase_pct`.
4. Dependencies: S5.
5. Preconditions: none.
6. Action: `duplicate` (same merchant or series key active on two accounts or as two series); `possible_overlap` (two active series in the same hub category); `price_increase` (latest amount ≥ previous × (1 + pct)); `trial_converted` (first paid charge in the last 60 days after a zero or trial record); `charged_after_cancellation` (a charge dated after the series was dismissed by cancellation); `lapsed`.
7. Autonomy: A2.
8. Timeout / retry: 5 s / none.
9. Idempotency: pure.
10. Postconditions: findings ordered by severity (charged after cancellation first).
11. Verification: each finding references existing series IDs and its numbers come from them.
12. Failure routes: none.
13. Escalation: none.
14. Evidence: findings with rule IDs.

**S7 — Word the suggestions**
1. Objective: short, useful, non-pushy text.
2. Capability: `nova.finance.advise.word_suggestions.llm` (fallback `.template`).
3. Input mapping: findings JSON (merchant, amounts, currency, change percentage; no account numbers).
4. Dependencies: S6.
5. Preconditions: approved provider.
6. Action / side effects: one LLM call.
7. Autonomy: A2.
8. Timeout / retry: 30 s / 1, then template.
9. Idempotency: no state change.
10. Postconditions: one suggestion per finding.
11. Verification: every number in the text appears in its finding; no new merchant names; no usage claims; ≤ 300 characters.
12. Failure routes: a failed check for any finding → template text for that finding.
13. Escalation: none.
14. Evidence: suggestion source per finding.

**S8 — Verify the report independently**
1. Objective: no wrong totals reach the user.
2. Capability: a separate verifier routine (not the analysis code path).
3. Input mapping: raw charges + report.
4. Dependencies: S7.
5. Preconditions: none.
6. Action: recompute cadences, monthly equivalents and totals from the raw charges; compare.
7. Autonomy: A2.
8. Timeout / retry: 10 s / none.
9. Idempotency: pure.
10. Postconditions: pass or a list of mismatches.
11. Verification: exact equality of decimals; same series set.
12. Failure routes: mismatch → drop the affected series from the totals, list them as "could not verify", and the run becomes `partially_completed`.
13. Escalation: none.
14. Evidence: verifier result.

**S9 — Deliver the digest**
1. Objective: a clear monthly message.
2. Capability: `nova.notify.owner.send`.
3. Input mapping: totals, top 5 findings, count of other series.
4. Dependencies: S8.
5. Preconditions: owner channel configured.
6. Action / side effects: one message: "Subscriptions: 5 active, about 45.57 USD a month (FX as of 30 Nov). Worth a look: …"; all strings escaped; no account numbers beyond the last 4 digits.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 2 on clear failure only.
9. Idempotency: key = report ID.
10. Postconditions: receipt.
11. Verification: receipt present.
12. Failure routes: failure → the report is still stored; `partially_completed`.
13. Escalation: none.
14. Evidence: message ID.

**S10 — Store the report and release**
1. Objective: a durable, versioned report.
2. Capability: `nova.finance.subscriptions.save_report`.
3. Input mapping: `audit_report`.
4. Dependencies: S8 (stored even if S9 fails).
5. Preconditions: verified report.
6. Action / side effects: insert the report; release the lock.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2.
9. Idempotency: report key.
10. Postconditions: report stored.
11. Verification: read-back equals the delivered numbers.
12. Failure routes: store fails → keep the result in the run record; `partially_completed`.
13. Escalation: none.
14. Evidence: report ID.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Read-only | zero writes except the report | scope audit log | zero | stop, report defect |
| Cadence | each listed cadence meets its rule | S8 recomputation | 100 % | move to `possible` |
| Totals | decimals equal on recomputation | S8 | exact | drop from totals, disclose |
| FX honesty | converted total only with as-of, source and non-expired status | report | holds | per-currency only |
| Number lock | every number in suggestions exists in the finding | S7 check | holds | template |
| No usage claims | no "unused" or "don't use" phrasing | S7 check | holds | template |
| Output schema | `audit_report` valid | validator | valid | fix; never fake values |

## J. Self-correction and recovery

- **Failure classes:** configuration, insufficient history, source unavailable, FX unavailable or expired, LLM failure, number-lock failure, verifier mismatch, delivery failure.
- **Permitted corrections:** continue with the available sources (disclosed); per-currency totals only; template wording; drop unverifiable series from totals.
- **Forbidden corrections:** estimating missing charges; using expired FX for a converted total; letting LLM text override computed numbers; writing to financial data.
- **Maximum correction cycles:** 1 per step.
- **Reverification required after correction:** yes — S8.
- **Stop conditions:** no data; kill switch.
- **Recovery or compensation:** a new report version supersedes a wrong one.
- **What must be reported:** missing sources, FX status, unverifiable series.

## K. Independent challenge

- **Assumptions to challenge:** two charges with the same merchant are the same service (a family plan and a personal plan may both be wanted); a price step is an increase and not a currency effect or tax change.
- **Alternative explanation:** a "duplicate" might be one charge recorded by both email and SMS that slipped through deduplication — S3 removes date ± 3 day matches, and the remainder is worded as "billed on two accounts", not "you pay twice".
- **Contradictory evidence:** a cancellation in the schedule with continued charges (that is a finding, not a contradiction to hide).
- **Could mislead:** a total that excludes subscriptions paid by a card not connected to the hub — the digest states which accounts were included.
- **Separate verifier:** S8 recomputes everything from raw charges by a separate routine.
- **Findings requiring disclosure:** excluded accounts, possible series, FX status.

## L. Human escalation

Not applicable for decisions — the recipe changes nothing and makes no consequential choice. The user acts on suggestions manually. The only escalations are operational (repeated delivery failure is shown in the report store), handled as in Section J.

## M. Completion conditions

Mandatory: `audit_report` valid and stored; totals verified; read-only scope held; FX disclosure correct; suggestions passed the number lock (or used templates); evidence references present.

- `completed` — report verified, stored and delivered.
- `partially_completed` — delivered or stored but not both, a source was missing, or some series could not be verified.
- `blocked` — invalid config or not enough history.
- `failed` — all sources unavailable or the read-only check failed.
- `cancelled` — kill switch or lock held.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome ("5 active subscriptions, about 45.57 USD/month, 2 things worth checking"); `audit_report`; step statuses S1–S10; verification results and failed checks; evidence references (report ID, FX snapshot ID, source row counts); approvals and scope (schedule enablement); side effects (report stored, message delivered); corrections performed; unresolved challenges; limitations (accounts included, FX status); escalation state; learning candidate. Never include full account numbers.

## O. Learning and revision

- **Outcome metrics:** series count, share of "possible" series, findings by type, user feedback ("not a subscription", "useful"), verifier mismatch rate, template fallback rate.
- **Evidence retained:** reports per policy.
- **Data minimisation:** reports hold merchants, amounts and series IDs; no message text.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}` — for example a new processor prefix to strip.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests plus the cadence fixture set.
- **Promotion approval:** publisher review plus Skill Port review.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (fixture history, 13 months) | correct series, totals and findings | UNVERIFIED on Skill Port; a monthly digest was observed in the reference implementation |
| Invalid input (bad currency code) | `blocked` | TBD |
| Missing input (no time zone) | `blocked` | TBD |
| Insufficient history (1 month) | `blocked` with explanation | TBD |
| Missing capability (payments-log binding absent) | continues with other sources; disclosed | TBD |
| Permission denied (`finance:hub:records:read` missing) | `blocked` or partial with disclosure | TBD |
| Unauthorized scope (account of another tenant in `accounts`) | ignored and reported | TBD |
| Tenant isolation | no other tenant's rows read | TBD |
| Read-only guarantee | zero writes besides the report | TBD |
| Duplicate across accounts | `duplicate` finding | TBD |
| Price increase | correct old/new/percent | TBD |
| Charge after cancellation | top finding | TBD |
| Expired FX | per-currency totals only | TBD |
| LLM invents a number | template used | TBD |
| Timeout (LLM) | template used; run completes | TBD |
| Duplicate / replay (scheduled run twice) | same report key; no second digest | TBD |
| False success (send OK without receipt) | `partially_completed` | TBD |
| Failed verification (injected arithmetic fault) | series dropped from totals, disclosed | TBD |
| Correction budget exhausted | safe stop | TBD |
| Escalation | not applicable — no consequential decisions (Section L) | n/a |
| Cancellation (kill switch) | `cancelled` | TBD |
| Rollback / compensation | not applicable — no financial state changes; a superseding report version is tested instead | TBD |
| Output schema and status | valid report | TBD |
| Compatibility / regression (schema change in hub records) | fixtures pass | TBD |

All tests use fictional histories in sandbox or rolled-back stores; never real accounts. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs and the report viewer — TBD (owner: publisher, with Skill Port).
- Annual subscriptions need at least one prior annual charge or a schedule entry to be detected.
- Subscriptions paid through accounts not connected to the hub are invisible.
- Category-based overlap depends on the hub's categories being set; otherwise only exact duplicates are found.
- No usage data exists, so the recipe never says a subscription is unused; it only asks the user to check.
