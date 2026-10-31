# Payment reminders

## A. Identity and ownership

- **Recipe ID:** `nova.finance.payment-reminders` (proposed until assigned by Skill Port)
- **Name:** Payment reminders
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** finance
- **Tags:** payments, reminders, alerts, due-dates, quiet-hours, deduplication, digest · Vedaxi task type: none (no supported task type covers this today)
- **Summary:** On a schedule, the agent reads the payment schedule and finds obligations that have crossed an alert threshold: newly detected, three days before, one day before, due today, or overdue. It sends one grouped digest to the user's own channel, respecting quiet hours. Each stage is marked as sent only after delivery is confirmed, so every stage fires once per cycle and no alert is lost.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** The obligation store produced by `nova.finance.upcoming-payments-radar`; an owner notification channel with delivery receipts; a runtime clock with the user's time zone. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — the scan, grouped digest, quiet hours, one-shot stage flags and send-then-confirm marking have run end-to-end in Nova's reference implementation (catalogue status: Built). Not every control described here has evidence yet (Section P lists the tests still needed). Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.finance.obligations` | Capability | The user's payment schedule | medium | UNVERIFIED |
| `nova.finance.obligations.advance_cycle` | Operation (update) | Move a recurring auto-charge obligation past its due date to the next cycle | low | UNVERIFIED |
| `nova.finance.obligations.alert_scan` | Operation (read) | Return open obligations with days-to-due and unsent stage flags; never marks anything | low | UNVERIFIED |
| `nova.finance.obligations.alert_mark` | Operation (update) | Set stage flags (sent or superseded) for the current cycle after delivery is confirmed | low | UNVERIFIED |
| `nova.notify.owner` | Capability | Deliver a message to the requesting user's own channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Send the digest and return a delivery receipt with a message ID | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.finance.obligations.alert_scan.default` — security-definer database function, read-only. Computes `days_to_due` in the user's time zone. Excludes paid, dismissed, series-dismissed, snoozed, needs-date and needs-review rows; returns only a count of the last two.
- `nova.finance.obligations.alert_mark.default` — sets flags only when the obligation's current `due_date` still equals the cycle being marked. If the cycle moved between scan and mark, the mark is a no-op.
- `nova.finance.obligations.advance_cycle.default` — recurring (weekly, monthly, quarterly, annual) auto-charge rows only. Adds whole periods until `due_date` ≥ today, resets the cycle stage flags and records history.
- `nova.notify.owner.send.plain` — plain text, the default. Every string from the schedule is treated as untrusted and sent unformatted.
- `nova.notify.owner.send.html` — channel HTML mode with all schedule strings HTML-escaped. Selectable only by operator configuration. If HTML mode is rejected, the digest is re-sent as plain text within the same run: this is the same channel, recipient and content, so it is not a provider or recipient change.

**Bindings:** `binding://nova/finance/obligations-advance-cycle`, `binding://nova/finance/obligations-alert-scan`, `binding://nova/finance/obligations-alert-mark`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: a scheduled automation-hub workflow, every 3 hours: scan → format → send through the chat channel → confirm. The schedule lives in PostgreSQL behind security-definer functions.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Obligation | `schema://nova/payment-reminders/obligation-alert-view@0.1.0` | the requesting user's tenant | read; update (`alert_state`, `status` derived field, `due_date` for auto-charge roll only) |
| Digest | `schema://nova/payment-reminders/digest@0.1.0` | the requesting user | create (sent once per run) |
| Mark | `schema://nova/payment-reminders/mark@0.1.0` | linked to one obligation cycle | create (applied as flag updates) |

**Authorization requirements (declared, never a grant):** `finance:obligations:read`, `finance:obligations:alert_mark`, `finance:obligations:advance_cycle`, `notify:owner`. Channel credentials: `secret_ref://owner-channel`; database role: `secret_ref://finance-db-role`.

**Policy / trust:** risk class `low` (messages only to the user; no money movement). Quiet hours and the one-shot rule are hard constraints. Merchant names come from emails, so they are untrusted text and are escaped.

## C. Intent and applicability

- **User problem:** A schedule only helps if the user is told in time, once, at a sensible hour, without a pile of duplicate pings.
- **Desired outcome:** For each obligation cycle the user gets at most one alert per stage, grouped into a single digest per run and never during quiet hours unless the money is due now.
- **Trigger or entry conditions:** A scheduled run (default every 3 hours), or "send me my payment reminders now" (the same rules apply, including the one-shot flags).
- **Applicable situations:** Any obligation in the schedule with a due date: renewals, bills, instalments, card dues.
- **Out-of-scope situations:** Paying or scheduling payments; nagging third parties; marking items paid (`nova.finance.auto-mark-bills-paid`); creating obligations (`nova.finance.upcoming-payments-radar`); reminders to anyone other than the requesting user.
- **Required inputs:** `timezone`, `owner_channel` (see Section D).
- **Prerequisites:** The obligation store installed and populated; the owner channel configured with delivery receipts.
- **Required capabilities:** `nova.finance.obligations`, `nova.notify.owner` (proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port; every binding is a dependency to resolve before activation.
- **Assumptions:** Verified in the reference implementation: threshold-based stages, one-shot flags, quiet hours with a due/overdue bypass, auto-charge items never escalating to due/overdue, and send-then-confirm. Not every control described here has evidence yet (Section P lists the tests still needed).
- **Ambiguity policy:** Use a documented safe default. If the time zone is missing, stop (`blocked`). Items without a date or with unresolved review are never alerted individually; only their count is mentioned.
- **Vedaxi task type:** none.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `timezone` | IANA zone | yes | user profile | — | `blocked`. |
| `owner_channel` | channel installation ref | yes | installation config | — | `blocked`. |
| `quiet_hours` | `{start:"HH:MM", end:"HH:MM"}` send window | no | user | `{"start":"08:00","end":"21:00"}` | Invalid: use the default and report it. |
| `critical_bypass_stages` | array subset of `due`, `overdue` | no | user | `["due","overdue"]` | Unknown values ignored. Empty means everything waits for the window. |
| `stage_windows` | `{new_days:1–60, t3_days:2–7, t1_days:1}` | no | operator | `{30,3,1}` | Out of range: clamp. |
| `max_items` | integer 5–50 | no | operator | 25 | Clamp. Overflow stays unmarked for the next run. |
| `include_review_count` | boolean | no | user | `true` | — |

```json
{"$id":"schema://nova/payment-reminders/config@0.1.0","type":"object","required":["timezone","owner_channel"],
 "properties":{"timezone":{"type":"string","minLength":3},"owner_channel":{"type":"string"},
 "quiet_hours":{"type":"object","required":["start","end"],"properties":{"start":{"type":"string","pattern":"^([01][0-9]|2[0-3]):[0-5][0-9]$"},"end":{"type":"string","pattern":"^([01][0-9]|2[0-3]):[0-5][0-9]$"}}},
 "critical_bypass_stages":{"type":"array","items":{"enum":["due","overdue"]}},
 "stage_windows":{"type":"object","properties":{"new_days":{"type":"integer","minimum":1,"maximum":60},"t3_days":{"type":"integer","minimum":2,"maximum":7},"t1_days":{"const":1}}},
 "max_items":{"type":"integer","minimum":5,"maximum":50},"include_review_count":{"type":"boolean"}}}
```

**Stage rules** (`d` = `due_date` − today in the user's time zone; each stage fires at most once per cycle, except `new`, which fires at most once per obligation):

| Stage | Condition | Auto-charge items |
|---|---|---|
| `new` | first scan after detection and `t3_days` < d ≤ `new_days` | yes ("renews automatically") |
| `t3` | 1 < d ≤ `t3_days` | yes ("will charge on …") |
| `t1` | d = 1 | yes |
| `due` | d = 0 | **no** — auto-charge never escalates |
| `overdue` | d < 0 and not paid | **no** — recurring rows roll to the next cycle (S2); one-off rows are left for paid tracking |

Only the most urgent unsent applicable stage is sent. Less urgent unsent stages for the same cycle are marked `superseded`, so a bill detected one day before it is due gets one `t1` alert, not `new` + `t3` + `t1`.

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `digest` | `schema://nova/payment-reminders/digest@0.1.0` | `digest_key`, `groups` | owner channel | financial; owner only | every line maps to a scanned item (Section I) |
| `reminder_run` | `schema://nova/payment-reminders/reminder-run@0.1.0` | `run_id`, `status`, `digest_sent`, `alerted` | run record | owner only | flags read back after marking |

```json
{"$id":"schema://nova/payment-reminders/digest@0.1.0","type":"object","required":["digest_key","groups"],
 "properties":{"digest_key":{"type":"string","pattern":"^[a-f0-9]{64}$"},
 "groups":{"type":"array","items":{"type":"object","required":["stage","items"],"properties":{
  "stage":{"enum":["overdue","due","t1","t3","new"]},
  "items":{"type":"array","items":{"type":"object","required":["obligation_id","merchant","due_date","auto_charge","cycle_due_date"],"properties":{
   "obligation_id":{"type":"string"},"merchant":{"type":"string","maxLength":120},
   "amount":{"type":["string","null"],"pattern":"^[0-9]{1,13}(\\.[0-9]{1,4})?$"},"currency":{"type":["string","null"],"pattern":"^[A-Z]{3}$"},
   "due_date":{"type":"string","format":"date"},"cycle_due_date":{"type":"string","format":"date"},"auto_charge":{"type":"boolean"},
   "account_hint":{"type":["string","null"],"pattern":"^.{0,40}··[0-9]{4}$"}}}}}}},
 "review_count":{"type":"integer","minimum":0},"overflow_count":{"type":"integer","minimum":0}}}
```

```json
{"$id":"schema://nova/payment-reminders/reminder-run@0.1.0","type":"object","required":["run_id","status","digest_sent","alerted"],
 "properties":{"run_id":{"type":"string"},"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},
 "digest_sent":{"type":"boolean"},"message_id":{"type":["string","null"]},
 "alerted":{"type":"array","items":{"type":"object","required":["obligation_id","cycle_due_date","stage"],"properties":{"obligation_id":{"type":"string"},"cycle_due_date":{"type":"string","format":"date"},"stage":{"enum":["new","t3","t1","due","overdue"]}}}},
 "superseded":{"type":"integer"},"deferred_quiet_hours":{"type":"integer"},"cycles_advanced":{"type":"integer"},"overflow":{"type":"integer"},
 "marks_confirmed":{"type":"boolean"},"limitations":{"type":"array","items":{"type":"string"}}}}
```

Supporting schemas (one per line):

```json
{"$id":"schema://nova/payment-reminders/local-date@0.1.0","type":"object","required":["today","timezone"],"properties":{"today":{"type":"string","format":"date"},"now_local":{"type":"string","format":"date-time"},"timezone":{"type":"string"},"stage_windows":{"type":"object"}}}
{"$id":"schema://nova/payment-reminders/obligation-alert-view@0.1.0","type":"object","required":["obligation_id","merchant","due_date","days_to_due","auto_charge","recurrence","alert_state"],"properties":{"obligation_id":{"type":"string"},"merchant":{"type":"string","maxLength":120},"amount":{"type":["string","null"],"pattern":"^[0-9]{1,13}(\\.[0-9]{1,4})?$"},"currency":{"type":["string","null"],"pattern":"^[A-Z]{3}$"},"due_date":{"type":"string","format":"date"},"days_to_due":{"type":"integer"},"auto_charge":{"type":"boolean"},"recurrence":{"enum":["weekly","monthly","quarterly","annual","one_off","unknown"]},"alert_state":{"type":"object","properties":{"new":{"type":"boolean"},"t3":{"type":"boolean"},"t1":{"type":"boolean"},"due":{"type":"boolean"},"overdue":{"type":"boolean"}}},"account_hint":{"type":["string","null"]},"review_count":{"type":"integer","minimum":0}}}
{"$id":"schema://nova/payment-reminders/mark@0.1.0","type":"object","required":["obligation_id","cycle_due_date","stage","kind"],"properties":{"obligation_id":{"type":"string"},"cycle_due_date":{"type":"string","format":"date"},"stage":{"enum":["new","t3","t1","due","overdue"]},"kind":{"enum":["sent","superseded"]},"message_id":{"type":["string","null"]}}}
{"$id":"schema://nova/payment-reminders/mark-result@0.1.0","type":"object","required":["applied","cycle_moved"],"properties":{"applied":{"type":"integer","minimum":0},"cycle_moved":{"type":"array","items":{"type":"string"}},"not_found":{"type":"array","items":{"type":"string"}}}}
{"$id":"schema://nova/payment-reminders/cycle-advance-result@0.1.0","type":"object","required":["advanced"],"properties":{"advanced":{"type":"array","items":{"type":"object","required":["obligation_id","old_due_date","new_due_date"],"properties":{"obligation_id":{"type":"string"},"old_due_date":{"type":"string","format":"date"},"new_due_date":{"type":"string","format":"date"}}}}}}
{"$id":"schema://nova/payment-reminders/delivery@0.1.0","type":"object","required":["outcome"],"properties":{"outcome":{"enum":["delivered","failed","ambiguous"]},"message_id":{"type":["string","null"]},"recipient":{"type":["string","null"]},"variant":{"enum":["plain","html"]},"error":{"type":["string","null"]}}}
```

**Example (fictional; run at 2026-11-10 09:00, America/New_York)**

Input: `{"timezone":"America/New_York","owner_channel":"chan_ex_01"}`

Digest text (plain): "Payments — 3 need attention. OVERDUE: Example Card statement, 230.00 USD, was due Mon 9 Nov (pay from Example Bank ··0142). DUE TODAY: Example Power & Water, 84.20 USD. IN 2 DAYS: Example Streaming, 12.99 USD, renews automatically Thu 12 Nov. 1 item needs a quick check in your schedule."

Output: `{"run_id":"run_0031","status":"completed","digest_sent":true,"message_id":"msg_ex_9001","alerted":[{"obligation_id":"obl_0003","cycle_due_date":"2026-11-09","stage":"overdue"},{"obligation_id":"obl_0005","cycle_due_date":"2026-11-10","stage":"due"},{"obligation_id":"obl_0007","cycle_due_date":"2026-11-12","stage":"t3"}],"superseded":2,"deferred_quiet_hours":0,"cycles_advanced":1,"overflow":0,"marks_confirmed":true,"limitations":[]}`

## E. Constraints and budgets

- **Hard constraints:** messages go only to the requesting user's channel; each stage fires at most once per cycle (`new` at most once per obligation); flags are set only after a delivery receipt; outside the quiet-hours window only `critical_bypass_stages` are sent; auto-charge items never get `due` or `overdue` alerts; amounts are shown as stored decimal strings with their explicit currency, or as "amount not stated" (never estimated); paid, dismissed, snoozed, needs-date and needs-review items are never alerted individually; no payment links or biller links are included; the agent never pays or schedules anything.
- **Soft constraints:** one digest per run; most urgent group first; short lines.
- **Resource and tenant scope:** the requesting user's obligations and owner channel only.
- **Data sources allowed:** the obligation store.
- **External transfers permitted:** the digest text to the owner channel provider.
- **Time limit:** 2 minutes per run.
- **Cost or token budget:** no LLM is used; one message per run.
- **Maximum steps:** 8 (S1–S8).
- **Maximum retries and correction cycles:** send 2 retries (on a clear failure only, never after an ambiguous outcome); mark 3 retries; correction cycles: 1.
- **Rate limits and concurrency:** one concurrent run per user; at most one digest per run; channel provider limits apply.
- **Retention and logging constraints:** run records keep obligation IDs and stages, not amounts; message text is not logged.
- **Prohibited actions:** messaging third parties; including full account numbers; re-sending a stage whose flag is set; clearing flags to "re-remind" without the user asking.
- **Stop conditions:** owner channel unavailable or unauthorized; kill switch; lock held.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Digest message (one per run) | at least one sendable item | Allowed (standing scope set when reminders are enabled) | delivery receipt with message ID |
| Read | Obligations (alert view) | own tenant | Allowed | — |
| Update | Obligation `alert_state` flags (sent / superseded) for the current cycle | delivery receipt received; cycle unchanged since the scan | Allowed | read-back shows the flags set |
| Update | Obligation `due_date` + cycle flags reset (recurring auto-charge past due only) | `auto_charge = true`, recurrence periodic | Allowed | new `due_date` ≥ today; history row written |
| Update | `status` derived values (`due_today`, `overdue`) | consistent with `days_to_due` | Allowed | read-back |
| Delete | any obligation, flag history or digest | — | Not Allowed | — |
| External side effect | message to the user's channel | within the window, or a bypass stage | Allowed | receipt |

- **Idempotency:** marks are keyed by `(obligation_id, cycle_due_date, stage)`; re-applying is a no-op. `digest_key` = SHA-256 of the sorted list of `obligation_id|cycle_due_date|stage`, passed to the channel as a client message key where the channel supports one.
- **Expected side effects:** one message to the user; flags set; recurring auto-charge cycles rolled forward.
- **Compensation:** the user can ask to re-send a stage (clears that one flag, with an audit record); a wrongly rolled cycle can be restored from history.
- **Irreversible:** a delivered message.
- **Ambiguous outcomes:** if `send` times out with no receipt, do not mark and do not resend in this run. The next run may send the same items again, so at most one duplicate is possible; the run record says so. A successful send followed by a failed mark is handled the same way.

## G. Autonomy and approval policy

- **Default autonomy level:** A2 — bounded autonomy (messages only to the user; one-shot rules).
- **Planning level:** A2.
- **Execution level:** A2.
- **Verification level:** A2.
- **Recovery level:** A2 — bounded retries, at-least-once delivery, no blind resend.
- **Approval required for:** enabling reminders (one-time standing scope); changing quiet hours or bypass stages; any re-send request.
- **Actions never permitted:** contacting billers or other people; paying; including payment links; alerting during quiet hours outside the bypass list; marking flags before delivery.
- **Conditions requiring clarification:** none at run time (configuration problems block the run).
- **Conditions requiring escalation:** the owner channel fails for 3 consecutive runs while an `overdue` or `due` item is pending (try the secondary owner channel if one is configured; otherwise show the failure in the schedule view).
- **Safe state after stop, cancellation or escalation:** no flags changed for unsent items; nothing marked without a receipt.

## H. Execution plan

**S1 — Validate configuration and take the run lock**
1. Objective: confirm the run may start.
2. Capability: none (local validation); runtime lock (UNVERIFIED binding).
3. Input mapping: installation config → `config@0.1.0`.
4. Dependencies: none.
5. Preconditions: reminders enabled; kill switch off.
6. Action / side effects: validate, acquire lock `reminders:<user>`, compute "today" and "now" in `timezone`.
7. Autonomy: A2; no approval.
8. Timeout / retry: 5 s / none.
9. Idempotency: lock prevents concurrent runs.
10. Postconditions: valid config; lock held; local date known.
11. Verification: schema valid; time zone resolves.
12. Failure routes: invalid → `blocked`; lock held → `cancelled`.
13. Escalation triggers: none.
14. Evidence: config hash, local date/time, lock ID.

**S2 — Roll recurring auto-charge cycles**
1. Objective: stop auto-charged renewals from appearing overdue.
2. Capability: `nova.finance.obligations.advance_cycle.default`.
3. Input mapping: today (local).
4. Dependencies: S1.
5. Preconditions: rows with `auto_charge = true`, periodic recurrence, `due_date` < today, not dismissed.
6. Action / side effects: add whole periods until `due_date` ≥ today; reset `t3`, `t1`, `due`, `overdue`; keep `new`; write history.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 2 retries.
9. Idempotency: the predicate (`due_date` < today) makes a repeat a no-op.
10. Postconditions: no periodic auto-charge row has `due_date` < today.
11. Verification: count of such rows = 0 after the call.
12. Failure routes: error → continue to S3 and disclose; affected rows are simply not alerted (auto-charge rows never get overdue alerts).
13. Escalation: none.
14. Evidence: advanced obligation IDs with old and new due dates.

**S3 — Scan for alertable items (read-only)**
1. Objective: the candidate list without changing anything.
2. Capability: `nova.finance.obligations.alert_scan.default`.
3. Input mapping: today, `stage_windows`.
4. Dependencies: S2.
5. Preconditions: none.
6. Action / side effects: read-only; returns open rows with `days_to_due`, flags and masked account hint, plus the needs-review/needs-date count.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: list of `obligation-alert-view` rows.
11. Verification: no returned row is paid, dismissed, snoozed (`snooze_until` ≥ today), needs-date or needs-review.
12. Failure routes: store unavailable → `failed`; nothing sent.
13. Escalation: none.
14. Evidence: scanned IDs and days-to-due.

**S4 — Select stages, apply quiet hours and caps**
1. Objective: decide exactly what to send now.
2. Capability: deterministic rules (part of the planner).
3. Input mapping: scan rows, stage table, `quiet_hours`, `critical_bypass_stages`, `max_items`, current local time.
4. Dependencies: S3.
5. Preconditions: none.
6. Action: for each row, find the most urgent applicable stage whose flag is unset (auto-charge rows cannot take `due` or `overdue`); mark lower unsent stages for supersession; outside the window keep only bypass stages; sort by urgency; cut at `max_items`.
7. Autonomy: A2.
8. Timeout / retry: 2 s / none.
9. Idempotency: pure function.
10. Postconditions: a send list and a supersede list; deferred and overflow counts.
11. Verification: no item appears twice; no flagged stage selected; quiet-hours rule holds.
12. Failure routes: nothing to send → skip to S8 with `completed` ("no reminders due").
13. Escalation: none.
14. Evidence: decision table (ID, stage, reason).

**S5 — Compose the digest**
1. Objective: one clear, safe message.
2. Capability: template rendering (no LLM).
3. Input mapping: send list, review count, overflow count.
4. Dependencies: S4.
5. Preconditions: a non-empty send list.
6. Action: group by stage (overdue, due, t1, t3, new); one line per item with merchant, amount + currency or "amount not stated", weekday + date, "renews automatically" for auto-charge, masked account hint; escape all strings for the chosen variant; compute `digest_key`.
7. Autonomy: A2.
8. Timeout / retry: 2 s / none.
9. Idempotency: `digest_key` is deterministic.
10. Postconditions: `digest@0.1.0` and rendered text ≤ the channel's length limit (split into numbered parts if longer).
11. Verification: every line maps to one send-list item; no URL present; no unescaped markup.
12. Failure routes: render error → fall back to the plain variant once → else `failed`.
13. Escalation: none.
14. Evidence: digest key and item count (not the text).

**S6 — Send to the user**
1. Objective: deliver the digest once.
2. Capability: `nova.notify.owner.send` (`plain` or `html`).
3. Input mapping: rendered text, `owner_channel`, `digest_key`.
4. Dependencies: S5.
5. Preconditions: channel authorized.
6. Action / side effects: one message (or numbered parts) to the user.
7. Autonomy: A2 within the standing scope.
8. Timeout / retry: 15 s; retry up to 2 times only on an explicit failure response; a timeout counts as ambiguous and is not retried.
9. Idempotency: `digest_key` as the client message key where supported.
10. Postconditions: a delivery receipt with message ID, or a clear failure, or an ambiguous outcome.
11. Verification: receipt present and its recipient is the owner channel.
12. Failure routes: clear failure → no marks, `failed` (retried next run); HTML rejected → plain once; ambiguous → no marks, `partially_completed` with a possible-duplicate note.
13. Escalation: 3 consecutive failed runs with due/overdue pending → escalate (Section L).
14. Evidence: message ID or the failure code.

**S7 — Confirm marks after delivery**
1. Objective: make each sent stage one-shot.
2. Capability: `nova.finance.obligations.alert_mark.default`.
3. Input mapping: sent marks `(obligation_id, cycle_due_date, stage, sent)` + supersede marks.
4. Dependencies: S6 with a receipt.
5. Preconditions: receipt present.
6. Action / side effects: set flags; update the derived `status` for due/overdue rows.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 3 retries (idempotent).
9. Idempotency: key `(obligation_id, cycle_due_date, stage)`; cycle guard.
10. Postconditions: flags set for every delivered item.
11. Verification: the function returns the applied count = the expected count, or a list of cycle-moved no-ops.
12. Failure routes: marks fail after retries → `partially_completed`; disclose that the next run may repeat these items.
13. Escalation: none.
14. Evidence: applied marks and no-ops.

**S8 — Verify and record the run**
1. Objective: an honest run record.
2. Capability: `nova.finance.obligations.alert_scan.default` (read-back).
3. Input mapping: alerted IDs.
4. Dependencies: S7 (or S4/S6 on the short paths).
5. Preconditions: none.
6. Action / side effects: re-scan the alerted IDs; store `reminder_run`; release the lock.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 1 retry.
9. Idempotency: run record key = `run_id`.
10. Postconditions: run record stored; lock released.
11. Verification: none of the alerted stages is still unset (unless a mark failure was disclosed).
12. Failure routes: mismatch → `partially_completed` with details.
13. Escalation: none.
14. Evidence: run record ID.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Input validity | config valid; time zone resolves | validator | valid | `blocked` |
| Scan is read-only | no row changed by S3 | row versions before and after | unchanged | stop, report defect |
| One-shot | no stage selected whose flag is already set | S4 decision table | none | drop the item, report |
| Supersession | at most one stage per cycle per run | S4 table | ≤ 1 | fix selection |
| Quiet hours | outside the window only bypass stages are sent | local time + send list | holds | do not send; defer |
| Auto-charge | no `due`/`overdue` for auto-charge items | send list | none | drop item |
| Send-then-confirm | marks applied only with a receipt | S6 receipt vs S7 call log | holds | revert marks, report |
| Content safety | no URLs; all strings escaped; masked account hints only | rendered text check | holds | re-render plain |
| Amount honesty | amount shown = stored decimal string + currency, or "amount not stated" | digest vs scan rows | exact | fix render |
| Output schema | `digest` and `reminder_run` valid | validator | valid | fix; never fake receipts |

A send call returning success proves only that the provider accepted the message. The receipt's message ID and recipient are what S7 depends on.

## J. Self-correction and recovery

- **Failure classes:** configuration, lock contention, store unavailable, render error, channel failure, ambiguous delivery, mark failure, cycle moved during the run.
- **Permitted corrections:** fall back from HTML to plain text on the same channel; retry clear send failures twice; retry idempotent marks; drop items whose cycle moved.
- **Forbidden corrections:** marking without a receipt; re-sending after an ambiguous outcome within the same run; sending to a different recipient; bypassing quiet hours for non-bypass stages; clearing flags to force a reminder.
- **Maximum correction cycles:** 1 per step; 2 per run.
- **Reverification required after correction:** yes — S5 content checks and S7 read-back.
- **Stop conditions:** budget exhausted; channel unauthorized; kill switch.
- **Recovery or compensation:** unsent items keep their flags unset and are retried next run; a wrongly rolled cycle is restored from history.
- **What must be reported:** deliveries, possible duplicates, deferred and overflow counts, mark failures.

## K. Independent challenge

- **Assumptions to challenge:** today's date in the user's time zone is right (the user may be travelling); an auto-charge item really auto-charges (a failed card means it is effectively overdue).
- **Alternative explanation:** an "overdue" item may already be paid but not yet reconciled; the digest says "if already paid, mark it paid" rather than asserting non-payment.
- **Contradictory evidence:** a payments-log entry for the same series after the due date (`nova.finance.auto-mark-bills-paid`), or a newer obligation for the same series.
- **Could mislead:** a quiet run that sent nothing because the schedule is stale. The run record includes the schedule's last update time, and the digest notes it if older than 24 hours.
- **Separate verifier:** the S8 read-back and the deterministic S4 rules are independent of the rendering step.
- **Findings requiring disclosure:** possible duplicate after an ambiguous send; deferred critical items; overflow.

## L. Human escalation

- **Triggers:** owner channel failing for 3 consecutive runs while a due/overdue item is pending; the channel authorization was revoked.
- **Why the agent cannot decide:** only the user can repair the channel or choose another one; switching to an unconfigured recipient is forbidden.
- **Information presented:** pending critical items (merchant, due date), the failure reason, the last successful delivery time.
- **Decision requested:** reconnect the channel, configure a secondary owner channel, or accept the schedule view as the only surface.
- **Options and consequences:** reconnecting resumes alerts; doing nothing means alerts keep queuing (flags stay unset).
- **Safe behaviour while waiting:** keep scanning; never mark unsent items.
- **If no response:** alerts stay queued; the schedule view shows an "alerts not delivered" banner.
- **Resume:** the next run after repair sends the most urgent unsent stage per item (supersession prevents a burst).

## M. Completion conditions

Mandatory: `reminder_run` valid; every alerted stage has a receipt and a confirmed mark (or the gap is disclosed); quiet-hours, one-shot and auto-charge rules held; no third-party contact; evidence references present.

- `completed` — the digest was delivered and all marks confirmed, or nothing was due.
- `partially_completed` — the digest was delivered but some marks failed, delivery was ambiguous, items overflowed, or S2 failed.
- `blocked` — invalid config, owner channel missing or unauthorized.
- `failed` — the store was unavailable or the send failed clearly (nothing marked; retried next run).
- `cancelled` — kill switch or lock held.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome ("3 reminders sent: 1 overdue, 1 due today, 1 in 2 days"); `reminder_run`; step statuses S1–S8; verification results and failed checks; evidence references (message ID, applied marks, advanced cycles); approvals and scope (standing scope ID); side effects (message delivered, flags set, cycles rolled) and reconciliation status; corrections performed; unresolved challenges; limitations; escalation state; learning candidate. Never include full account numbers, credentials or message text in shared logs.

## O. Learning and revision

- **Outcome metrics:** alerts per stage, duplicate rate (stages delivered twice), deferred-by-quiet-hours count, time from `due` alert to mark-paid, user re-send and snooze requests.
- **Evidence retained:** run records and mark history, per the retention policy.
- **Data minimisation:** no message text or amounts in run records; obligation IDs only.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}` — for example "users snooze `new` alerts 80 % of the time → shorten `new_days`".
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review; stage-window or quiet-hours default changes are versioned.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (sandbox schedule, test channel) | one grouped digest; flags set after receipt | UNVERIFIED on Skill Port; observed in the reference implementation |
| Invalid input (bad quiet-hours format) | default used, reported | TBD |
| Missing input (no channel) | `blocked` | TBD |
| Missing capability (scan binding absent) | `blocked`; nothing sent | TBD |
| Permission denied (`notify:owner` missing) | `blocked` | TBD |
| Unauthorized scope (another tenant's obligation ID in a mark) | rejected | TBD |
| Tenant isolation | scan never returns other tenants' rows | TBD |
| One-shot | second run with no changes sends nothing | TBD |
| Supersession (item detected at d = 1) | only `t1` sent; `new` and `t3` superseded | TBD |
| Quiet hours | at 23:00 a `t3` waits; an `overdue` is sent | TBD |
| Auto-charge past due | rolled to next cycle; no overdue alert | TBD |
| Snoozed / needs-review items | not alerted; review count shown | TBD |
| Timeout on send (ambiguous) | no marks; next run may repeat once; disclosed | TBD |
| Duplicate / replay (S7 re-applied) | no-op | TBD |
| Cycle moved between scan and mark | mark is a no-op | TBD |
| False success (provider OK, no message ID) | treated as ambiguous; no marks | TBD |
| Failed verification (flag not set after mark) | `partially_completed` | TBD |
| Correction budget exhausted | stops; unsent items keep flags unset | TBD |
| Escalation (channel failing 3 runs with overdue pending) | user told via the schedule view / secondary channel | TBD |
| Cancellation (kill switch) | `cancelled`; nothing sent | TBD |
| Compensation (user asks to re-send a stage) | one flag cleared with audit; resent next run | TBD |
| Output schema and status | valid objects; correct status | TBD |
| Compatibility / regression (channel API change) | plain variant still delivers | TBD |

All tests use a sandbox schedule with fictional obligations, a test channel owned by the tester and rolled-back transactions; never real reminders to real people. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, schema registry and channel adapter — TBD (owner: publisher, with Skill Port).
- Should `due`/`overdue` also wait for the quiet-hours window? The default is to send them (owner: user).
- Should an overdue manual bill get repeated nudges (for example weekly) rather than one alert? The default is one (owner: publisher).
- A failed auto-charge cannot be detected from the schedule alone; it shows up only if a failed-payment email reaches the radar.
- At-least-once delivery means a single duplicate digest is possible after an ambiguous send.
- The time zone comes from the user profile; travel is not detected.
