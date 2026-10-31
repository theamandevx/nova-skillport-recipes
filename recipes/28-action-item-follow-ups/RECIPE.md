# Action item follow-ups

## A. Identity and ownership

- **Recipe ID:** `nova.calendar.action-item-follow-ups` (proposed until assigned by Skill Port)
- **Name:** Action item follow-ups
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** productivity
- **Tags:** meetings, minutes, action-items, reminders, evening-digest, read-only · Vedaxi task type: none (not one of the seven task types Vedaxi supports today)
- **Summary:** Each evening at the user's chosen local time, the agent reads the open action items from the user's meeting minutes, groups them into overdue, due today or tomorrow, no due date and waiting on others, and sends one plain-text reminder to the user only. It never edits, closes or reassigns an item and never messages anyone else.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** The minutes store with owner-tagged action items (contracts shared with recipe 25, Meeting minutes); an append-only nudge log; an owner notification channel; a runtime scheduler that can fire per user at a local time. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — in Nova's reference implementation the evening owner digest pattern (consolidate pending items, skip silently when empty, owner-only plain-text delivery with a dashboard link) and the minutes store with owner-tagged action items both ran. Due-date grouping depends on the optional due dates from recipe 25 and is unverified. Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.minutes.store` | Capability | Read minutes sessions and their action items | medium | UNVERIFIED |
| `nova.minutes.store.list_open_actions` | Operation (read) | Return open action items in the user's tenant within a lookback window | low | UNVERIFIED |
| `nova.minutes.store.get` | Operation (read) | Read one minutes session to confirm an item's source | low | UNVERIFIED |
| `nova.minutes.followups` | Capability | Keep an append-only record of reminders sent, for dedupe | low | UNVERIFIED |
| `nova.minutes.followups.record_nudge` | Operation (create, append-only, conditional) | Append a nudge entry keyed by user and local date; on conflict, return the existing entries | low | UNVERIFIED |
| `nova.notify.owner` | Capability | Deliver a message to the requesting user's own channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Send the evening reminder to the user | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.minutes.store.list_open_actions.default` — read-only; only the default variant is supported. Binding `binding://nova/minutes/open-actions` (UNVERIFIED), protocol: internal database function behind the runtime; mode: sync.
- `nova.minutes.store.get.default` — read-only; only the default variant is supported.
- `nova.minutes.followups.record_nudge.default` — append-only, conditional on the key (`user_id`, `local_date`, `kind`); entries are never updated or deleted by this recipe.
- `nova.notify.owner.send.default` — plain text only (no markdown parse mode), owner channel only. There is no variant that can address anyone other than the user, and no fallback to a different channel without the user's configuration.
- The composition step is deterministic (no LLM), so counts and dates are always computed, never generated.

**Bindings:** `binding://nova/minutes/open-actions`, `binding://nova/minutes/store-get`, `binding://nova/minutes/nudge-log`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. The scheduler is a runtime dependency with no proposed binding (`TBD`). Reference implementation: a scheduled workflow on an automation hub that calls security-definer database functions, composes the message in code and sends it to the user's chat with a link to the minutes dashboard.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Action item | `schema://nova/action-item-follow-ups/action-item@0.1.0` | the user's tenant | read |
| Minutes session | `schema://nova/action-item-follow-ups/minutes-ref@0.1.0` | the user's tenant | read |
| Nudge log entry | `schema://nova/action-item-follow-ups/nudge@0.1.0` | the user | create (append-only) |
| Owner message | `schema://nova/action-item-follow-ups/delivery@0.1.0` | the user's channel | create |

**Authorization requirements (declared, never a grant):** `minutes:read`, `followups:append`, `notify:owner`. No write scope on minutes or action items is requested. The channel credential lives in the runtime as `secret_ref://owner-channel`.

**Policy / trust:** risk class `low` (read-only, owner-only message). The user enabling the reminder is a narrowly bounded pre-approved scope: one message per local date, to the user only. Quiet hours apply.

## C. Intent and applicability

- **User problem:** Action items agreed in meetings get lost; the user wants one short evening reminder of what is still open, without editing anything for them.
- **Desired outcome:** At most one accurate, plain-text reminder per day listing the user's open items by urgency and what others owe them, each linked to its minutes.
- **Trigger or entry conditions:** The scheduler fires at `send_at_local` in the user's time zone, the reminder is enabled, and no reminder has been recorded for that local date.
- **Applicable situations:** Users whose meetings produce minutes with owner-tagged action items (recipes 25, 26, 27).
- **Out-of-scope situations:** Marking items done, editing, reassigning or deleting items; messaging action-item owners other than the user (drafting a follow-up to a colleague is handed to a comms recipe that requires approval); creating tasks in other tools; reminders outside the configured window.
- **Required inputs:** `user_id`, `timezone`, `owner_aliases` (see Section D).
- **Prerequisites:** minutes store reachable; owner channel configured; scheduler able to fire at local times.
- **Required capabilities:** `nova.minutes.store`, `nova.minutes.followups`, `nova.notify.owner` (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port; the scheduler binding is `TBD`.
- **Assumptions:** Verified in the reference implementation: owner-tagged action items in minutes; an evening digest that skips when empty and sends plain text to the owner. Assumed: due dates and confidence values are present when recipe 25 supplies them; items without them still appear.
- **Ambiguity policy:** Use a documented safe default where one exists (send time 18:00, lookback 30 days). Stop when the time zone is missing or invalid — due-date grouping depends on it and must not be guessed.
- **Vedaxi task type:** none of the seven supported today.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `user_id` | string | yes | runtime | — | Missing: `blocked`. |
| `timezone` | IANA zone name | yes | user settings | — | Missing or invalid: `blocked` with "set your time zone"; never guessed. |
| `owner_aliases` | array of strings (names, emails, "Me"), 1–20 | yes | user settings; personal data | — | Empty: `blocked`; items cannot be attributed to the user. |
| `send_at_local` | `HH:MM` 24-hour | no | user | `18:00` | Invalid: use the default and tell the user once. |
| `catch_up_window_min` | integer 0–180 | no | operator | 120 | A run later than this after `send_at_local` is skipped (`missed_window`). |
| `quiet_hours` | `{start:"HH:MM", end:"HH:MM"}` | no | user | 21:00–08:00 | A run inside quiet hours is skipped, not delayed overnight. |
| `lookback_days` | integer 1–90 | no | user | 30 | Clamp; older open items are counted, not listed. |
| `max_items_per_group` | integer 1–20 | no | user | 5 | Clamp; the rest are summarised as "and N more". |
| `include_waiting_on_others` | boolean | no | user | `true` | — |
| `portal_base_url` | URL | no | operator | — | Absent: items show the minutes ID instead of a link. |
| `enabled` | boolean | yes | user | `false` until the user turns it on | `false`: run ends `cancelled` before any read. |

```json
{"$id":"schema://nova/action-item-follow-ups/config@0.1.0","type":"object","required":["user_id","timezone","owner_aliases","enabled"],
 "properties":{"user_id":{"type":"string"},"timezone":{"type":"string","minLength":3},"owner_aliases":{"type":"array","minItems":1,"maxItems":20,"items":{"type":"string"}},
 "send_at_local":{"type":"string","pattern":"^([01][0-9]|2[0-3]):[0-5][0-9]$"},"catch_up_window_min":{"type":"integer","minimum":0,"maximum":180},
 "quiet_hours":{"type":"object","properties":{"start":{"type":"string"},"end":{"type":"string"}}},"lookback_days":{"type":"integer","minimum":1,"maximum":90},
 "max_items_per_group":{"type":"integer","minimum":1,"maximum":20},"include_waiting_on_others":{"type":"boolean"},"portal_base_url":{"type":"string","format":"uri"},"enabled":{"type":"boolean"}}}
```

```json
{"$id":"schema://nova/action-item-follow-ups/action-item@0.1.0","type":"object","required":["item_id","minutes_id","task","status"],
 "properties":{"item_id":{"type":"string"},"minutes_id":{"type":"string"},"meeting_title":{"type":"string"},"meeting_date":{"type":"string","format":"date"},
 "task":{"type":"string"},"owner":{"type":["string","null"]},"due_date":{"type":["string","null"],"format":"date"},
 "confidence":{"enum":["high","medium","low",null]},"status":{"enum":["open","done","dropped"]}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `followup_digest` | object `schema://nova/action-item-follow-ups/digest@0.1.0` | `local_date`, `status`, `sent`, `skipped_reason`, `counts` | the user's channel (message text) and the run record | personal data; owner only | counts recomputed from the item list; item IDs matched to the read set (Section I) |

```json
{"$id":"schema://nova/action-item-follow-ups/digest@0.1.0","type":"object","required":["local_date","status","sent","skipped_reason","counts"],
 "properties":{"local_date":{"type":"string","format":"date"},"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},
 "sent":{"type":"boolean"},"skipped_reason":{"enum":["none","empty","already_sent","quiet_hours","missed_window","disabled"]},
 "groups":{"type":"object","properties":{"overdue":{"type":"array","items":{"type":"string"}},"due_soon":{"type":"array","items":{"type":"string"}},
 "no_due_date":{"type":"array","items":{"type":"string"}},"owner_unclear":{"type":"array","items":{"type":"string"}},"waiting_on_others":{"type":"array","items":{"type":"string"}}}},
 "counts":{"type":"object","properties":{"overdue":{"type":"integer"},"due_soon":{"type":"integer"},"no_due_date":{"type":"integer"},"due_later":{"type":"integer"},"owner_unclear":{"type":"integer"},"waiting_on_others":{"type":"integer"},"older_than_lookback":{"type":"integer"}}},
 "low_confidence_item_ids":{"type":"array","items":{"type":"string"}},"message_id":{"type":["string","null"]},"nudge_ref":{"type":["string","null"]}}}
```

Other schemas (compact; groups hold `item_id` values):

```json
{"$id":"schema://nova/action-item-follow-ups/minutes-ref@0.1.0","type":"object","required":["minutes_id","title","meeting_date"],"properties":{"minutes_id":{"type":"string"},"title":{"type":"string"},"meeting_date":{"type":"string","format":"date"},"url":{"type":["string","null"],"format":"uri"}}}
{"$id":"schema://nova/action-item-follow-ups/nudge@0.1.0","type":"object","required":["user_id","local_date","kind","at"],"properties":{"user_id":{"type":"string"},"local_date":{"type":"string","format":"date"},"kind":{"enum":["claimed","sent","skipped_empty","skipped_other","failed"]},"item_ids":{"type":"array","items":{"type":"string"}},"message_id":{"type":["string","null"]},"at":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/action-item-follow-ups/delivery@0.1.0","type":"object","required":["message_id","delivered_at"],"properties":{"message_id":{"type":"string"},"delivered_at":{"type":"string","format":"date-time"},"channel":{"type":"string"}}}
```

**Example (fictional)**

Input: `{"user_id":"usr_0001","timezone":"America/New_York","owner_aliases":["Jane Doe","Jane","Me","jane@example.com"],"send_at_local":"18:00","enabled":true,"portal_base_url":"https://app.example.com"}`

Message (plain text, Tuesday 2026-11-03):

```
Open action items - Tue 3 Nov
Overdue (1)
1. Send the revised budget sheet - due Mon 2 Nov - Budget check-in https://app.example.com/minutes/min_0001
Due today or tomorrow (1)
2. Book the workshop room - due Wed 4 Nov - Planning sync https://app.example.com/minutes/min_0002
No due date (1)
3. Share the design notes (low confidence - check the minutes) - Planning sync https://app.example.com/minutes/min_0002
Waiting on others (1)
4. John Roe: review the contract draft - due Thu 5 Nov - Budget check-in https://app.example.com/minutes/min_0001
Mark items done on the minutes page. To chase John, ask me to draft a follow-up (needs your approval).
```

Output: `{"local_date":"2026-11-03","status":"completed","sent":true,"skipped_reason":"none","groups":{"overdue":["ai_0001"],"due_soon":["ai_0004"],"no_due_date":["ai_0005"],"owner_unclear":[],"waiting_on_others":["ai_0002"]},"counts":{"overdue":1,"due_soon":1,"no_due_date":1,"due_later":0,"owner_unclear":0,"waiting_on_others":1,"older_than_lookback":0},"low_confidence_item_ids":["ai_0005"],"message_id":"msg_0001","nudge_ref":"nudge_0001"}`

## E. Constraints and budgets

- **Hard constraints:** read-only on minutes and action items — never edit, close, reassign or delete; message the user only, never an action-item owner or attendee; at most one reminder per user per local date; never send inside quiet hours or after the catch-up window; plain text only; nothing invented — every listed item exists in the read set, and counts are computed, not generated; tenant scope is the user's own minutes.
- **Soft constraints:** keep the message under 3,500 characters; most urgent groups first; omit empty groups.
- **Resource and tenant scope:** the user's minutes, action items and nudge log only.
- **Data sources allowed:** the minutes store; the user's settings.
- **External transfers permitted:** the reminder text to the user's own channel provider.
- **Time limit:** whole run ≤ 2 minutes.
- **Cost or token budget:** no LLM calls; storage and messaging costs only (`TBD` per operator).
- **Maximum steps:** 9 (S1–S9).
- **Maximum retries and correction cycles:** reads: 2 retries; send: 2 retries after reconciliation; correction cycles: 1.
- **Rate limits and concurrency:** one run per user per local date; runs for different users may proceed in parallel within provider limits.
- **Retention and logging constraints:** nudge log keeps item IDs and message IDs, not task text; message text is not logged; retention per the operator's policy (`TBD`).
- **Prohibited actions:** writing to minutes or action items; contacting third parties; sending a second reminder for the same date; using markdown parse modes.
- **Stop conditions:** reminder disabled; time zone or aliases missing; already sent today; inside quiet hours or outside the window; nothing open.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Nudge log entry (`claimed`, `sent`, `skipped_empty`, `skipped_other`, `failed`) | one `claimed` per user and local date | Allowed (append-only) | entry readable on conflict replay |
| Read | Action items, minutes sessions | user's tenant; within `lookback_days` | Allowed | — |
| Update | any | — | Not Allowed | — |
| Delete | any | — | Not Allowed | — |
| External side effect | reminder message to the user | enabled; claimed; outside quiet hours; non-empty | Allowed under the user's enablement | delivery receipt |
| External side effect | any message to anyone other than the user | — | Not Allowed (hand off to a comms recipe with approval) | — |

- **Idempotency:** claim key = (`user_id`, `local_date`, `claimed`); message key = hash(`user_id`, `local_date`). A replayed run finds the claim and stops or reconciles.
- **Expected side effects:** one message to the user per local date with open items; nudge log entries.
- **Compensation:** none needed for reads; a sent message cannot be recalled; nudge entries are append-only and are not rolled back.
- **Irreversible:** the delivered message.
- **Ambiguous outcomes:** a `claimed` entry without `sent` or `failed` means a previous run stopped mid-way; look up the message key with the channel before any resend. If the lookup is impossible, do not resend; record `failed` and disclose.

## G. Autonomy and approval policy

- **Default autonomy level:** A2 — bounded autonomy within the scope the user enabled.
- **Planning level:** A2.
- **Execution level:** A2 for reads, the nudge log and the owner-only message.
- **Verification level:** A2.
- **Recovery level:** A2 for bounded retries; A0 for anything outside scope.
- **Approval required for:** turning the reminder on (the user's own setting); any follow-up message to a colleague (handed to a comms recipe, approved per message).
- **Actions never permitted:** editing, closing, reassigning or deleting items; messaging anyone but the user; a second reminder for the same date; sending in quiet hours.
- **Conditions requiring clarification:** missing or invalid time zone; empty owner aliases.
- **Conditions requiring escalation:** owner channel unavailable for 3 consecutive days; read errors suggesting another tenant's data.
- **Safe state after stop, cancellation or escalation:** nothing sent, nothing changed in the minutes store, the outcome recorded in the nudge log.

Enabling the reminder covers one owner-only message per local date. It does not cover messages to anyone else or any change to action items.

## H. Execution plan

**S1 — Check trigger and window**
1. Objective: run only when a reminder is due now.
2. Capability: runtime scheduler (binding `TBD`); local validation.
3. Input mapping: `config@0.1.0`, current UTC time → local time in `timezone`.
4. Dependencies: none.
5. Preconditions: scheduler fired for this user.
6. Action / side effects: validate the config; compute `local_date`; check enabled, catch-up window and quiet hours; no side effects.
7. Autonomy: A2; no approval.
8. Timeout / retry: 5 s / none.
9. Idempotency: not applicable.
10. Postconditions: `local_date` fixed for the whole run, or a skip reason.
11. Verification: config valid against `config@0.1.0`; local time computed with the IANA zone (daylight-saving aware).
12. Failure routes: disabled → `cancelled` (`disabled`); time zone or aliases missing → `blocked`; quiet hours or missed window → `completed` with `sent = false` and the reason.
13. Escalation triggers: none.
14. Evidence: local date, local time, window decision.

**S2 — Claim today's reminder**
1. Objective: guarantee at most one reminder per local date.
2. Capability: `nova.minutes.followups.record_nudge` (kind `claimed`).
3. Input mapping: `user_id`, `local_date`.
4. Dependencies: S1.
5. Preconditions: within the window.
6. Action / side effects: conditional append of a `claimed` entry; on conflict the existing entries are returned instead.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries (safe: conditional).
9. Idempotency: claim key (`user_id`, `local_date`, `claimed`).
10. Postconditions: this run owns the claim, or a previous run's entries are known.
11. Verification: the returned entry carries this run's ID, or the conflict result lists prior entries.
12. Failure routes: prior `sent` or `skipped_*` → stop, `completed` (`already_sent`); prior `claimed` without an outcome → reconcile via the message key (Section F) before continuing.
13. Escalation: none.
14. Evidence: claim entry ID or conflict entries.

**S3 — Read open action items**
1. Objective: get every open item relevant to the user.
2. Capability: `nova.minutes.store.list_open_actions`.
3. Input mapping: `user_id` (tenant), `lookback_days`.
4. Dependencies: S2.
5. Preconditions: claim owned.
6. Action / side effects: read-only query for items with `status = open` from minutes in the window, plus a count of older open items.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: a list of `action-item@0.1.0` and `older_than_lookback`.
11. Verification: every item's tenant equals the user's; every item validates against the schema.
12. Failure routes: read fails after retries → `failed`, record `failed` in the nudge log; a foreign-tenant item → discard all, `failed`, escalate.
13. Escalation: foreign-tenant data.
14. Evidence: item IDs and count (no task text in logs).

**S4 — Classify by owner and due date**
1. Objective: put each item in exactly one group.
2. Capability: none (deterministic code).
3. Input mapping: items, `owner_aliases`, `local_date`, `include_waiting_on_others`.
4. Dependencies: S3.
5. Preconditions: items read.
6. Action / side effects: owner matches an alias (case-insensitive, exact name or email) → the user's item; user's items go to `overdue` (due before today), `due_soon` (today or tomorrow), `no_due_date`, or are counted as `due_later`; a named non-user owner → `waiting_on_others`; a null or unmatched-ambiguous owner → `owner_unclear`; `confidence = low` → labelled.
7. Autonomy: A2.
8. Timeout / retry: 5 s / none.
9. Idempotency: deterministic.
10. Postconditions: groups with counts.
11. Verification: sum of group counts plus `due_later` equals the number of items read.
12. Failure routes: count mismatch → recompute once → else `failed`.
13. Escalation: none.
14. Evidence: group counts.

**S5 — Verify items against their minutes**
1. Objective: list only items that still exist, are open and link to real minutes.
2. Capability: `nova.minutes.store.get`.
3. Input mapping: distinct `minutes_id` values of listed items (up to `max_items_per_group` per group).
4. Dependencies: S4.
5. Preconditions: groups built.
6. Action / side effects: read each referenced session; confirm the item is present and still open; build its link.
7. Autonomy: A2.
8. Timeout / retry: 10 s per session / 1 retry.
9. Idempotency: read-only.
10. Postconditions: every listed item has a confirmed source and link (or minutes ID).
11. Verification: item text in the digest equals the stored task text; status still `open`.
12. Failure routes: item closed since S3 → drop it and recount; session unreadable → keep the item with "source unavailable" and mark the run `partially_completed`.
13. Escalation: none.
14. Evidence: minutes IDs confirmed.

**S6 — Skip when empty**
1. Objective: send nothing when nothing is open.
2. Capability: `nova.minutes.followups.record_nudge` (kind `skipped_empty`).
3. Input mapping: group counts.
4. Dependencies: S5.
5. Preconditions: verified groups.
6. Action / side effects: if every listed group is empty (and `due_later` is 0), append `skipped_empty` and stop; otherwise continue.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: keyed by `user_id`, `local_date`.
10. Postconditions: either a `skipped_empty` entry, or the run proceeds to compose.
11. Verification: no message is sent when total open = 0.
12. Failure routes: append fails → stop anyway (nothing to send) and report.
13. Escalation: none.
14. Evidence: skip entry ID.

**S7 — Compose the reminder**
1. Objective: a short, accurate plain-text message.
2. Capability: none (deterministic template).
3. Input mapping: groups, counts, links, `max_items_per_group`, `local_date`.
4. Dependencies: S6.
5. Preconditions: at least one open item.
6. Action / side effects: order groups overdue → due soon → no due date → owner unclear → waiting on others; number items; show due dates in the user's locale; add "and N more" and the due-later and older counts; strip markdown control characters (asterisks, underscores, square brackets, backticks); end with the read-only footer and the hand-off line for chasing others.
7. Autonomy: A2.
8. Timeout / retry: 5 s / none.
9. Idempotency: deterministic for the same inputs.
10. Postconditions: message ≤ 3,500 characters.
11. Verification: every numbered line maps to an item ID; counts in headings equal group sizes; no markdown control characters remain.
12. Failure routes: over the length limit → lower `max_items_per_group` by 1 and recompose (once) → else truncate with "open the minutes page for the rest".
13. Escalation: none.
14. Evidence: message hash, line → item ID map.

**S8 — Send to the user**
1. Objective: deliver one reminder to the user only.
2. Capability: `nova.notify.owner.send`.
3. Input mapping: message text, message key.
4. Dependencies: S7.
5. Preconditions: claim held; still outside quiet hours (re-checked).
6. Action / side effects: one message on the user's channel, no parse mode.
7. Autonomy: A2, covered by the user's enablement.
8. Timeout / retry: 15 s / 2 retries, each after checking the message key.
9. Idempotency: message key = hash(`user_id`, `local_date`).
10. Postconditions: a delivery receipt.
11. Verification: receipt received; recipient equals the configured owner channel.
12. Failure routes: channel rejects the text → resend once as the shortest form (counts only plus link); still failing → `failed`.
13. Escalation: channel unavailable 3 days running → tell the user through the portal.
14. Evidence: message ID.

**S9 — Record the outcome**
1. Objective: close the day's entry so no second reminder is sent.
2. Capability: `nova.minutes.followups.record_nudge` (kind `sent` or `failed`).
3. Input mapping: `user_id`, `local_date`, listed item IDs, message ID.
4. Dependencies: S8.
5. Preconditions: send attempted.
6. Action / side effects: append the outcome entry.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 3 retries.
9. Idempotency: keyed by `user_id`, `local_date`, kind.
10. Postconditions: the day has exactly one outcome entry.
11. Verification: the conflict replay returns `claimed` plus one outcome.
12. Failure routes: append fails after retries → result `partially_completed`; the next run reconciles by message key.
13. Escalation: none.
14. Evidence: nudge entry ID.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Input validity | config matches `config@0.1.0`; time zone resolves | validator | valid | `blocked` |
| Window | send time within the catch-up window and outside quiet hours (checked at S1 and S8) | clock and settings | both true | skip with reason |
| Once per day | one `claimed` and at most one `sent` per user and local date | nudge log | true | stop; never resend |
| Tenant | every item belongs to the user's tenant | item records | 100 % | discard, `failed`, escalate |
| Read-only | no write call to minutes or items during the run | runtime call log | zero writes | incident, escalate |
| Completeness | group counts plus `due_later` = items read | S4 report | equal | recompute, else `failed` |
| No invention | every message line maps to a stored, open item | S5/S7 maps | 100 % | drop the line, recount |
| Recipient | only the configured owner channel | delivery receipt | match | stop, escalate |
| Format | no markdown control characters; ≤ 3,500 characters | message check | true | sanitise, recompose |
| Output schema | `followup_digest` valid | validator | valid | fix fields; never fake values |

A successful send proves only that a message reached the channel, not that the list was correct; correctness rests on the completeness and no-invention checks.

## J. Self-correction and recovery

- **Failure classes:** validation, missing settings, missed window, quiet hours, transient read error, foreign-tenant data, count mismatch, item closed mid-run, channel rejection, ambiguous prior run.
- **Permitted corrections:** retry reads; recompute groups once; drop items closed since the read; shorten the message; reconcile a prior claim by message key.
- **Forbidden corrections:** send a second reminder for the same date; send in quiet hours; write to items; message anyone else; hide low-confidence labels; pad counts.
- **Maximum correction cycles:** 1 per step; 3 per run.
- **Reverification required after correction:** yes — re-run the completeness and no-invention checks after any recompute or shortening.
- **Stop conditions:** correction budget exhausted; tenant check fails.
- **Recovery or compensation:** none needed beyond recording `failed`; the next scheduled day runs normally.
- **What must be reported:** skipped reasons, dropped items, shortened messages, delivery failures.

## K. Independent challenge

- **Assumptions to challenge:** that an alias match is really the user (two people named Jane); that "open" in the store is current; that a due date extracted from minutes was meant as a deadline.
- **Alternative approach:** a weekly digest may suit users with few meetings; offer it after 5 consecutive empty days (setting change by the user only).
- **Contradictory evidence:** later minutes that report an item as done while the earlier item is still open — list it under its group with "may be done (see later minutes)" rather than hiding it.
- **Could mislead:** low-confidence items shown without a label; "waiting on others" read as a request to chase.
- **Separate verifier:** completeness and no-invention checks are rule-based against the store, independent of the composer.
- **Findings requiring disclosure:** owner-unclear items, low-confidence items, items older than the lookback, unreadable sources.

## L. Human escalation

- **Triggers:** missing time zone or aliases; foreign-tenant data; owner channel unavailable 3 days running; repeated count mismatches.
- **Why the agent cannot decide:** identity, tenant boundaries and channel configuration belong to the user or operator.
- **Information presented:** the failing check, local date, counts (no task text in operator views).
- **Decision requested:** set the time zone or aliases; fix the channel; pause the reminder.
- **Options and consequences:** pausing stops all reminders until re-enabled; fixing settings resumes on the next scheduled day.
- **Safe behaviour while waiting:** no reminders sent; minutes untouched.
- **If no response:** the reminder stays paused for that user; nothing is retried outside the window.
- **Resume:** the next scheduled run re-validates the config and authorization from S1.

## M. Completion conditions

Mandatory: `followup_digest` valid; once-per-day rule held; no write to minutes or items; every listed item verified; only the user messaged; skip reasons and limitations disclosed; evidence references present.

- `completed` — reminder delivered and recorded, or a correct silent skip (`empty`, `already_sent`, `quiet_hours`, `missed_window`) recorded.
- `partially_completed` — delivered but an item source was unreadable, the message was truncated, or the outcome could not be recorded.
- `blocked` — time zone, aliases or a required binding missing, or authorization denied.
- `failed` — reads failed after retries, a tenant or completeness check failed, or delivery failed after retries.
- `cancelled` — the reminder is disabled, or the user turned it off before the send.

A skipped day is reported with `sent = false` and its reason, never as a delivered reminder.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome; `followup_digest`; step statuses S1–S9; verification results and failed checks; evidence references (claim and outcome nudge IDs, message ID, item IDs); approvals and scope (the user's enablement, owner-only); side effects (message sent or not) and reconciliation status; corrections performed; unresolved challenges; limitations; escalation state; learning candidate. Never include task text in operator logs or any channel other than the user's.

## O. Learning and revision

- **Outcome metrics:** reminders sent per week, empty-skip rate, items per reminder, overdue share, owner-unclear share, delivery failure rate, items later marked done within 48 hours.
- **Evidence retained:** run record and nudge entries (IDs only), per the retention policy.
- **Data minimisation:** no task text in metrics or logs.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}`.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review; no revision may add a write scope or a non-owner recipient.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (sandbox tenant, fictional minutes) | one plain-text reminder with correct groups and links | UNVERIFIED on Skill Port; digest pattern observed in the reference |
| Invalid input (bad time zone, `25:00` send time) | `blocked` / default time used with notice | TBD |
| Missing input (no aliases) | `blocked` with instructions | TBD |
| Missing capability (minutes store binding absent) | `blocked` with a dependency message; nothing sent | TBD |
| Permission denied (`minutes:read` missing) | `blocked`; nothing sent | TBD |
| Unauthorized scope (attempted write to an item) | rejected by the runtime; incident logged | TBD |
| Tenant isolation (another tenant's items in the read) | discarded; `failed`; escalated | TBD |
| Empty (no open items) | no message; `skipped_empty` recorded | TBD |
| Quiet hours and missed window | no message; reason recorded | TBD |
| Daylight-saving change day | correct local date and send time | TBD |
| Timeout / retry on send | message key checked before resend; one message | TBD |
| Duplicate / replay (scheduler fires twice) | second run finds the claim and stops | TBD |
| False success (send OK, item closed after the read) | closed item dropped at S5; counts correct | TBD |
| Failed verification (count mismatch) | recomputed once, else `failed` | TBD |
| Correction budget exhausted | stops safely; nothing extra sent | TBD |
| Escalation (channel down 3 days) | user told through the portal | TBD |
| Cancellation (reminder disabled before send) | `cancelled`; nothing sent | TBD |
| Rollback / compensation | Not applicable — no writes to roll back; a sent message cannot be recalled; nudge entries are append-only by design | — |
| Output schema and status | valid `followup_digest`, correct status | TBD |
| Compatibility / regression (minutes without `due_date` or `confidence`) | items appear under no due date, unlabelled | TBD |

All tests use fictional minutes in a sandbox tenant or rolled-back transactions; never real users' channels. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, the scheduler binding and the schema registry — TBD (owner: publisher, with Skill Port).
- Due-date grouping depends on recipe 25 supplying `due_date` and `confidence`; items without them all land in "no due date" (owner: publisher).
- Alias matching is exact (names and emails); nicknames or misspellings in transcripts fall into "owner unclear".
- The reminder is read-only by design; marking items done happens on the minutes page, outside this recipe.
- Chasing a colleague needs a separate comms recipe with per-message approval; not provided here.
- Retention period for nudge entries — TBD (owner: operator).
- Message length limits differ by channel; the 3,500-character cap is a conservative default.
