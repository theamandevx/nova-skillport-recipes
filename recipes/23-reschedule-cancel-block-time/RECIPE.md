# Reschedule, cancel or block time

## A. Identity and ownership

- **Recipe ID:** `nova.calendar.reschedule-cancel-block-time` (proposed until assigned by Skill Port)
- **Name:** Reschedule, cancel or block time
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** productivity
- **Tags:** calendar, reschedule, cancel, block-time, busy-block, time-zones, guest-notification, human-approval, idempotent · Vedaxi task type: none of the seven supported today
- **Summary:** On an explicit request from the calendar owner, the agent moves or cancels an existing booking (guests are notified, so each change needs approval) or adds or removes an owner-only busy block (no third party is told, so no approval is needed). Every change is idempotent and verified against the calendar state, not just the tool's return value.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** A calendar store that is the master busy-store (bookings plus busy blocks), a slot engine, an external calendar provider (for example Google Calendar) reached from a single propagation point, an approval gateway, and an owner notification channel. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — reschedule, cancel, block and unblock ran in Nova's reference implementation, including database-level propagation that moved or deleted the external calendar event (verified with database-only bookings that have no external event, so no email fired). Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.calendar.booking` | Capability | Read and change the owner's bookings | high | UNVERIFIED |
| `nova.calendar.booking.list` | Operation (read) | MCP tools `list_bookings` / `upcoming_bookings`: find bookings by ID, guest or time | low | UNVERIFIED |
| `nova.calendar.booking.reschedule` | Operation (update + history row + external side effect) | MCP tool `reschedule_booking`: move a booking, keeping its ID and meeting link | high | UNVERIFIED |
| `nova.calendar.booking.cancel` | Operation (update + external side effect) | MCP tool `cancel_booking`: set status `cancelled`; returns the external event ID | high | UNVERIFIED |
| `nova.calendar.availability` | Capability | Free time and calendar views | low | UNVERIFIED |
| `nova.calendar.availability.get_slots` | Operation (read) | MCP tool `get_slots`: free start times for a duration | low | UNVERIFIED |
| `nova.calendar.availability.list_calendar` | Operation (read) | Unified bookings and busy blocks for a range; conflict checks | low | UNVERIFIED |
| `nova.calendar.busy_sync` | Capability | Freshness of synced busy sources | low | UNVERIFIED |
| `nova.calendar.busy_sync.status` | Operation (read) | Last successful sync per source (calendar sync, invite sync) | low | UNVERIFIED |
| `nova.calendar.blocks` | Capability | Owner-only busy blocks | low | UNVERIFIED |
| `nova.calendar.blocks.block` | Operation (create) | MCP tool `block_time`: add a manual busy block | low | UNVERIFIED |
| `nova.calendar.blocks.unblock` | Operation (delete, manual blocks only) | Remove a manual busy block | low | UNVERIFIED |
| `nova.calendar.invites` | Capability | Keep the external calendar event in step, with attendee notifications | high | UNVERIFIED |
| `nova.calendar.invites.push` | Operation (external side effect) | Update or delete the external event; invoked only by the single propagation point | high | UNVERIFIED |
| `nova.approvals.gateway` | Capability | Ask the owner to approve one specific action | medium | UNVERIFIED |
| `nova.approvals.gateway.request` | Operation (create + message to owner) | Send Approve / Deny as one-tap token links | medium | UNVERIFIED |
| `nova.approvals.gateway.status` | Operation (read) | Return `pending`, `approved`, `denied` or `expired` | low | UNVERIFIED |
| `nova.notify.owner` | Capability | Deliver a message to the owner's own channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Send the result to the owner | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.calendar.availability.get_slots.default` — owner agent. May use a minimum notice of 0 for the owner's own changes; the past-time guard always applies. Selected for this recipe.
- `nova.calendar.availability.get_slots.public` — external callers (guest self-serve page, third-party MCP clients); minimum-notice and host overrides are stripped server-side. Not eligible here and never a fallback for `.default` (different constraints).
- `nova.calendar.invites.push.send-all` — production: the calendar provider emails every attendee about the move or cancellation. Selected only when the approval says guests will be notified.
- `nova.calendar.invites.push.send-none` — tests, sandboxes, and an owner-approved silent change; no emails. Never selected silently for a real reschedule or cancel, because guests would not learn of the change. No fallback between the two variants: a failed `send-all` is reported, not retried as `send-none`.
- `nova.calendar.booking.list.default`, `nova.calendar.booking.reschedule.default`, `nova.calendar.booking.cancel.default`, `nova.calendar.availability.list_calendar.default`, `nova.calendar.busy_sync.status.default`, `nova.calendar.blocks.block.default`, `nova.calendar.blocks.unblock.default`, `nova.approvals.gateway.request.default`, `nova.approvals.gateway.status.default`, `nova.notify.owner.send.default` — only the default variant is supported.

**Bindings:** `binding://nova/calendar/list-bookings`, `binding://nova/calendar/reschedule-booking`, `binding://nova/calendar/cancel-booking`, `binding://nova/calendar/get-slots`, `binding://nova/calendar/calendar-range`, `binding://nova/calendar/busy-sync-status`, `binding://nova/calendar/block-time`, `binding://nova/calendar/unblock-time`, `binding://nova/calendar/invite-push`, `binding://nova/approvals/request`, `binding://nova/approvals/status`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: calendar tables in PostgreSQL behind security-definer functions, exposed as MCP tools by an automation hub (for example n8n). Reschedule and cancel queue one outbound HTTP call from the database layer to the propagation webhook, so every caller (agent, guest self-serve page, portal, MCP client) is covered once and nothing double-fires.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Booking | `schema://nova/reschedule-cancel-block-time/booking@0.1.0` | the owner's tenant | read; update (`start`, `end`, `status` only) |
| Booking history row | same schema, status `rescheduled` | linked to one booking | create (by reschedule only) |
| Busy block (manual) | `schema://nova/reschedule-cancel-block-time/block@0.1.0` | the owner | create, read, delete |
| Busy block (synced) | same schema, `source` ≠ `manual` | the owner, owned by the sync | read only |
| External calendar event | reference only (`external_event_id`) | the owner's provider account | update / delete via the propagation point only |
| Guest self-manage token | opaque, never shown to the agent | linked to one booking | transferred by reschedule (never copied) |
| Approval | `schema://nova/reschedule-cancel-block-time/approval@0.1.0` | the owner | create, read |

**Authorization requirements (declared, never a grant):** `calendar:bookings:read`, `calendar:bookings:write` (reschedule and cancel), `calendar:availability:read`, `calendar:blocks:write`, `approvals:request`, `notify:owner`. The external-calendar write scope is held by the propagation point, not by the agent: `secret_ref://calendar-provider-oauth`, `secret_ref://propagation-hook-token`.

**Policy / trust:** risk class `high` for reschedule and cancel (guests are emailed); `low` for block and unblock (owner-only). Requires the per-change approval in S5 for every guest-notifying change.

## C. Intent and applicability

- **User problem:** Plans change. The owner needs to move or cancel a meeting, or keep time free, without opening the calendar and without forgetting to tell guests.
- **Desired outcome:** The right booking moved or cancelled exactly once with guests notified, or the right busy block added or removed, and a truthful confirmation.
- **Trigger or entry conditions:** The owner explicitly asks to reschedule, cancel, block, unblock, or says "I'm busy from X to Y".
- **Applicable situations:** Moving a meeting to another free slot; cancelling a meeting; blocking focus or travel time; clearing a manual block; clearing a time range that already contains meetings.
- **Out-of-scope situations:** Creating new bookings (recipe 22); guests changing their own booking through the tokenized self-serve page (separate flow — this recipe must respect its changes, see S6); removing synced blocks (remove them at the source calendar or invite); editing another person's calendar; bulk cancellations of more than 10 bookings in one run; recurring-series edits.
- **Required inputs:** `action` plus the fields that action needs (see Section D).
- **Prerequisites:** The calendar store, slot engine and propagation point are installed for the owner's tenant; the owner's IANA time zone is configured; an approval channel works.
- **Required capabilities:** `nova.calendar.booking`, `nova.calendar.availability`, `nova.calendar.busy_sync`, `nova.calendar.blocks`, `nova.calendar.invites` (indirectly, via the propagation point), `nova.approvals.gateway`, `nova.notify.owner` (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port; every binding is a dependency to resolve before activation.
- **Assumptions:** Verified in the reference implementation: reschedule keeps the booking ID and link and moves the external event; cancel deletes the external event; propagation fires from one point; blocks change the slot engine's output. Assumed: the target runtime offers equivalent tools and a provider that emails attendees on update and delete.
- **Ambiguity policy:** Ask. If the booking, the new time, its time zone or the block range is unclear, ask one clarifying question; never guess which meeting to move.
- **Vedaxi task type:** None of the seven supported today.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `action` | enum `reschedule`, `cancel`, `block`, `unblock`, `busy_range` | yes | owner | — | Ask once; never infer a cancel. |
| `booking_ref` | string: booking ID, guest name or a start time | for `reschedule`, `cancel` | owner; personal data | — | Ask. Several matches: list them (owner-zone time and guest) and ask which. |
| `new_start` | ISO 8601 date-time with offset, or a local time plus `time_zone` | for `reschedule` | owner | — | Ask. Past or inside a daylight-saving gap: reject and ask. |
| `time_zone` | IANA zone, for example `America/New_York` | no | owner profile | the owner's configured zone | Unknown zone: ask. Never assume UTC silently. |
| `range_start`, `range_end` | ISO 8601 date-times with offset | for `block`, `busy_range` | owner | — | `range_start >= range_end` (empty or negative range): reject and ask. |
| `block_id` | string | for `unblock` | owner | — | Ask, or list today's manual blocks. |
| `title` | string ≤ 80 chars | no | owner; owner-only | "Busy" | Truncate and tell the owner. |
| `reason` | string ≤ 300 chars | no | owner; shared with guests only if the owner says so | none | Never invent a reason. |
| `send_updates` | enum `all`, `none` | no | owner | `all` | `none` only on the owner's explicit choice in the approval, or in tests. |

```json
{"$id":"schema://nova/reschedule-cancel-block-time/request@0.1.0","type":"object","required":["action"],
 "properties":{"action":{"enum":["reschedule","cancel","block","unblock","busy_range"]},
  "booking_ref":{"type":"string","minLength":2},"new_start":{"type":"string","format":"date-time"},
  "time_zone":{"type":"string"},"range_start":{"type":"string","format":"date-time"},
  "range_end":{"type":"string","format":"date-time"},"block_id":{"type":"string"},
  "title":{"type":"string","maxLength":80},"reason":{"type":"string","maxLength":300},
  "send_updates":{"enum":["all","none"]}},
 "allOf":[{"if":{"properties":{"action":{"const":"reschedule"}}},"then":{"required":["booking_ref","new_start"]}},
  {"if":{"properties":{"action":{"const":"cancel"}}},"then":{"required":["booking_ref"]}},
  {"if":{"properties":{"action":{"enum":["block","busy_range"]}}},"then":{"required":["range_start","range_end"]}},
  {"if":{"properties":{"action":{"const":"unblock"}}},"then":{"required":["block_id"]}}]}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `change_result` | object `schema://nova/reschedule-cancel-block-time/change-result@0.1.0` | `status`, `action`, `changes` | the owner's channel and the run record | personal data; owner only | each change re-read from the calendar (S8) |

```json
{"$id":"schema://nova/reschedule-cancel-block-time/change-result@0.1.0","type":"object",
 "required":["status","action","changes"],
 "properties":{"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},
  "action":{"enum":["reschedule","cancel","block","unblock","busy_range"]},
  "changes":{"type":"array","maxItems":11,"items":{"type":"object","required":["kind","target_id","outcome"],
   "properties":{"kind":{"enum":["reschedule","cancel","block","unblock","keep"]},"target_id":{"type":"string"},
    "outcome":{"enum":["applied","already_in_state","declined","failed","not_attempted"]},
    "start_utc":{"type":"string","format":"date-time"},"end_utc":{"type":"string","format":"date-time"},
    "start_owner_local":{"type":"string"},"start_guest_local":{"type":["string","null"]},
    "meeting_url":{"type":["string","null"],"format":"uri"},"approval_id":{"type":["string","null"]},
    "propagation":{"enum":["sent","not_applicable","not_repeated","failed","unknown"]},
    "guests_notified":{"type":"integer","minimum":0}}}},
  "notes":{"type":"array","items":{"type":"string"}}}}
```

Other schemas (compact; all intervals half-open with `start_utc < end_utc`):

```json
{"$id":"schema://nova/reschedule-cancel-block-time/booking@0.1.0","type":"object","required":["booking_id","status","start_utc","end_utc","version"],"properties":{"booking_id":{"type":"string"},"title":{"type":"string"},"status":{"enum":["confirmed","cancelled","rescheduled"]},"start_utc":{"type":"string","format":"date-time"},"end_utc":{"type":"string","format":"date-time"},"guests":{"type":"array","items":{"type":"object","required":["name","email"],"properties":{"name":{"type":"string"},"email":{"type":"string","format":"email"}}}},"guest_time_zone":{"type":["string","null"]},"meeting_url":{"type":["string","null"],"format":"uri"},"external_event_id":{"type":["string","null"]},"version":{"type":"integer","minimum":1}}}
{"$id":"schema://nova/reschedule-cancel-block-time/block@0.1.0","type":"object","required":["block_id","start_utc","end_utc","source","synced"],"properties":{"block_id":{"type":"string"},"start_utc":{"type":"string","format":"date-time"},"end_utc":{"type":"string","format":"date-time"},"title":{"type":"string","maxLength":80},"source":{"enum":["manual","calendar_sync","invite_sync"]},"synced":{"type":"boolean"}}}
{"$id":"schema://nova/reschedule-cancel-block-time/approval@0.1.0","type":"object","required":["approval_id","action","target_id","scope_hash","decision"],"properties":{"approval_id":{"type":"string"},"action":{"enum":["reschedule","cancel"]},"target_id":{"type":"string"},"scope_hash":{"type":"string"},"booking_version":{"type":"integer"},"expires_at":{"type":"string","format":"date-time"},"decision":{"enum":["pending","approved","denied","expired"]}}}
{"$id":"schema://nova/reschedule-cancel-block-time/approval-request@0.1.0","type":"object","required":["action","target_id","summary","scope_hash"],"properties":{"action":{"enum":["reschedule","cancel"]},"target_id":{"type":"string"},"summary":{"type":"string","maxLength":1200},"scope_hash":{"type":"string"},"booking_version":{"type":"integer"},"expires_in_min":{"type":"integer","maximum":60}}}
{"$id":"schema://nova/reschedule-cancel-block-time/approval-ref@0.1.0","type":"object","required":["approval_id"],"properties":{"approval_id":{"type":"string"}}}
{"$id":"schema://nova/reschedule-cancel-block-time/booking-query@0.1.0","type":"object","minProperties":1,"properties":{"booking_id":{"type":"string"},"guest_name":{"type":"string"},"start_utc":{"type":"string","format":"date-time"},"from_utc":{"type":"string","format":"date-time"},"to_utc":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/reschedule-cancel-block-time/reschedule-input@0.1.0","type":"object","required":["booking_id","new_start_utc","expected_version","approval_id","idempotency_key"],"properties":{"booking_id":{"type":"string"},"new_start_utc":{"type":"string","format":"date-time"},"expected_version":{"type":"integer"},"approval_id":{"type":"string"},"send_updates":{"enum":["all","none"]},"idempotency_key":{"type":"string"}}}
{"$id":"schema://nova/reschedule-cancel-block-time/cancel-input@0.1.0","type":"object","required":["booking_id","expected_version","approval_id"],"properties":{"booking_id":{"type":"string"},"expected_version":{"type":"integer"},"approval_id":{"type":"string"},"reason":{"type":"string","maxLength":300},"send_updates":{"enum":["all","none"]}}}
{"$id":"schema://nova/reschedule-cancel-block-time/slot-query@0.1.0","type":"object","required":["duration_min","from_utc","to_utc"],"properties":{"duration_min":{"type":"integer","minimum":15,"maximum":240},"from_utc":{"type":"string","format":"date-time"},"to_utc":{"type":"string","format":"date-time"},"exclude_booking_id":{"type":"string"}}}
{"$id":"schema://nova/reschedule-cancel-block-time/slot-list@0.1.0","type":"object","required":["slots"],"properties":{"slots":{"type":"array","items":{"type":"object","required":["start_utc","end_utc"],"properties":{"start_utc":{"type":"string","format":"date-time"},"end_utc":{"type":"string","format":"date-time"}}}}}}
{"$id":"schema://nova/reschedule-cancel-block-time/range@0.1.0","type":"object","required":["start_utc","end_utc"],"properties":{"start_utc":{"type":"string","format":"date-time"},"end_utc":{"type":"string","format":"date-time"},"title":{"type":"string","maxLength":80},"client_key":{"type":"string"}}}
{"$id":"schema://nova/reschedule-cancel-block-time/calendar-items@0.1.0","type":"object","required":["items"],"properties":{"items":{"type":"array","items":{"type":"object","required":["kind","id","start_utc","end_utc"],"properties":{"kind":{"enum":["booking","block"]},"id":{"type":"string"},"start_utc":{"type":"string","format":"date-time"},"end_utc":{"type":"string","format":"date-time"},"source":{"type":"string"},"synced":{"type":"boolean"}}}}}}
{"$id":"schema://nova/reschedule-cancel-block-time/block-ref@0.1.0","type":"object","required":["block_id"],"properties":{"block_id":{"type":"string"}}}
{"$id":"schema://nova/reschedule-cancel-block-time/sync-query@0.1.0","type":"object","properties":{"sources":{"type":"array","items":{"enum":["calendar_sync","invite_sync"]}}}}
{"$id":"schema://nova/reschedule-cancel-block-time/sync-status@0.1.0","type":"object","required":["sources"],"properties":{"sources":{"type":"array","items":{"type":"object","required":["source","last_success_at"],"properties":{"source":{"type":"string"},"last_success_at":{"type":["string","null"],"format":"date-time"},"last_error":{"type":["string","null"]}}}}}}
{"$id":"schema://nova/reschedule-cancel-block-time/push-input@0.1.0","type":"object","required":["booking_id","external_event_id","op","send_updates"],"properties":{"booking_id":{"type":"string"},"external_event_id":{"type":"string"},"op":{"enum":["update","delete"]},"start_utc":{"type":"string","format":"date-time"},"end_utc":{"type":"string","format":"date-time"},"send_updates":{"enum":["all","none"]}}}
{"$id":"schema://nova/reschedule-cancel-block-time/push-result@0.1.0","type":"object","required":["external_event_id","outcome"],"properties":{"external_event_id":{"type":"string"},"outcome":{"enum":["sent","not_found","failed"]},"attendees_notified":{"type":"integer","minimum":0}}}
{"$id":"schema://nova/reschedule-cancel-block-time/delivery@0.1.0","type":"object","required":["message_id","delivered"],"properties":{"message_id":{"type":"string"},"delivered":{"type":"boolean"}}}
```

**Example (fictional)**

Input: `{"action":"reschedule","booking_ref":"bk_0001","new_start":"2026-11-06T09:30:00","time_zone":"America/New_York"}`

Output: `{"status":"completed","action":"reschedule","changes":[{"kind":"reschedule","target_id":"bk_0001","outcome":"applied","start_utc":"2026-11-06T14:30:00Z","end_utc":"2026-11-06T15:00:00Z","start_owner_local":"Fri 6 Nov 09:30 America/New_York","start_guest_local":"Fri 6 Nov 14:30 Europe/London","meeting_url":"https://meet.example.com/abc-defg-hij","approval_id":"apr_0001","propagation":"sent","guests_notified":1}],"notes":["Same booking ID and link; John Roe (john@example.org) was emailed by the calendar provider."]}`

## E. Constraints and budgets

- **Hard constraints:** explicit owner request; per-change approval (S5) for every reschedule or cancel, one approval per booking; block and unblock are owner-only and never notify a third party; every time is converted explicitly from a named IANA zone to UTC and shown back in the owner's and the guest's zones; past-time guard (no new start before now); the new slot must be free per `get_slots` and `list_calendar`; reject `range_start >= range_end` (a zero-length block is an empty half-open range and blocks nothing); never remove a synced block; never delete booking records (cancel is a status change); never call `invites.push` directly — the single propagation point does it; never copy a guest self-manage token (transfer only); never test against a real calendar.
- **Soft constraints:** prefer the slot nearest the requested time; keep the owner's title and reason short; offer at most 3 alternative slots.
- **Resource and tenant scope:** the owner's bookings, blocks and approvals only.
- **Data sources allowed:** the calendar store, the slot engine, sync status, the owner's message.
- **External transfers permitted:** the booking's time, title and guest list to the external calendar provider, solely to update or delete the existing event.
- **Time limit:** whole run ≤ 45 minutes including approval waits; each tool call ≤ 20 s.
- **Cost or token budget:** LLM use limited to parsing and the change summary; per-run token cap `TBD` (operator).
- **Maximum steps:** 9 (S1–S9); `busy_range` repeats S3–S6 per conflict, at most 10 conflicts.
- **Maximum retries and correction cycles:** read steps: 2 retries; mutating steps: 0 blind retries (reconcile first); correction cycles: 1 per step, 3 per run.
- **Rate limits and concurrency:** one run per booking at a time; provider rate limits apply.
- **Retention and logging constraints:** run records keep IDs, times and approval references; logs never contain tokens, guest self-manage tokens or full guest emails in shared channels. Retention period `TBD`.
- **Prohibited actions:** moving a meeting outside the approved time; adding or removing guests; changing the meeting link; sending free-text messages to guests beyond the owner's approved reason.
- **Stop conditions:** approval denied or expired; booking changed since approval; new slot taken; synced block targeted; propagation failure (stop further changes and report).

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Booking history row (status `rescheduled`) | only as part of an approved reschedule | Approval Required (S5) | exactly one history row for the move |
| Create | Manual busy block | explicit owner request; `start < end` | Allowed | block visible in `list_calendar`; slots excluded |
| Read | Bookings, blocks, slots, sync status, approvals | own tenant only | Allowed | — |
| Update | Booking `start`, `end` (reschedule) | approval valid; version unchanged; slot free | Approval Required (S5) | same booking ID and link at the new time |
| Update | Booking `status` → `cancelled` | approval valid; booking `confirmed` | Approval Required (S5) | status `cancelled`; slot freed |
| Update | Guest self-manage token moved to the live row | inside the reschedule transaction | covered by S5 | token resolves to the live row only |
| Delete | Manual busy block | explicit owner request; `source = manual` | Allowed | block absent; slots offered again |
| Delete | Synced block, booking records | — | Not Allowed | — |
| External side effect | external event moved or deleted; attendees emailed (`send-all`) | via the single propagation point | Approval Required (S5) | provider returns success for exactly one request |
| External side effect | result message to the owner | run finished | Allowed | delivery receipt |

- **Idempotency:** reschedule key = hash(`run_id`, `booking_id`, `new_start`, `approval_id`); a replay returns the existing result. Cancel of an already-cancelled booking returns the current state with `propagation: not_repeated` and sends no second email. Block client key = hash(`owner_id`, `range_start`, `range_end`, `run_id`) returns the same block on replay. Unblock of an absent block returns `already_in_state`.
- **Expected side effects:** guests receive an update or cancellation email from the calendar provider; the slot engine's output changes.
- **Compensation:** a reschedule can be moved back (a new approved reschedule, which emails guests again); a block can be removed; a cancel cannot be undone — a new booking (recipe 22) would send a fresh invite.
- **Irreversible:** emails already sent to guests; a cancelled external event.
- **Ambiguous outcomes:** if an apply call times out, read the booking by ID (S8) before anything else; never repeat a reschedule or cancel on an unknown outcome.

## G. Autonomy and approval policy

- **Default autonomy level:** A1 — supervised execution.
- **Planning level:** A2 (resolve the target, check slots, draft the summary).
- **Execution level:** A1 for reschedule and cancel (after S5); A2 for block and unblock on the owner's explicit request.
- **Verification level:** A2.
- **Recovery level:** A1 — no repeated change without reconciliation, and a new approval if the booking changed.
- **Approval required for:** each reschedule and each cancel (booking, old and new times in both zones, guests to be emailed, notification mode); choosing `send_updates: none` for a real booking.
- **Actions never permitted:** changing a booking without approval; removing synced blocks; deleting booking records; adding or removing guests; disclosing the owner's other meetings to guests; testing on a real calendar.
- **Conditions requiring clarification:** ambiguous booking; local time without a resolvable zone; a daylight-saving gap or fold; `busy_range` conflicts with no decision.
- **Conditions requiring escalation:** booking changed since approval (for example by the guest's self-serve page); propagation failed; duplicate live rows for one booking ID; an orphaned external event.
- **Safe state after stop, cancellation or escalation:** no half-applied change (each change is one transaction), nothing retried, the owner told exactly what changed and what did not.

Approval covers one change to one booking at one booking version. It does not cover later moves, other bookings, or the same booking after it changed.

## H. Execution plan

**S1 — Validate the request**
1. Objective: classify the action and validate its fields.
2. Capability: none (local validation).
3. Input mapping: owner message → `request@0.1.0`; relative dates ("tomorrow 9:30") resolved against now in `time_zone`.
4. Dependencies: none.
5. Preconditions: an explicit owner request.
6. Action / side effects: parse; convert every time from its named zone to UTC; no side effects.
7. Autonomy: A2; no approval.
8. Timeout / retry: 5 s / none.
9. Idempotency: not applicable.
10. Postconditions: a schema-valid request with UTC instants and the source zone recorded.
11. Verification: JSON Schema passes; `new_start` ≥ now; `range_start < range_end`; times in a daylight-saving gap or fold flagged.
12. Failure routes: missing or ambiguous field → ask once → `blocked`; empty or negative range → reject and ask; gap time → ask for another time.
13. Escalation triggers: none.
14. Evidence: validated request; original phrase → resolved UTC instants.

**S2 — Resolve the target**
1. Objective: exactly one booking (reschedule, cancel), one manual block (unblock), or the conflict set (block, busy_range).
2. Capability: `nova.calendar.booking.list`; `nova.calendar.availability.list_calendar`.
3. Input mapping: `booking_ref` → query; `[range_start, range_end)` → range.
4. Dependencies: S1.
5. Preconditions: request valid.
6. Action / side effects: read-only. Ranges are half-open, so a meeting ending exactly at `range_start` is not a conflict.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries on transient errors.
9. Idempotency: read-only.
10. Postconditions: one booking with status and version; or one block with its source; or a conflict list (possibly empty).
11. Verification: exactly one match; owner's tenant; booking `confirmed` for reschedule or cancel; block `source = manual` for unblock.
12. Failure routes: several matches → ask which; none → `blocked`; already cancelled → `already_in_state`, stop with `completed` and no email; synced block → refuse and say it must be removed at its source → `blocked`.
13. Escalation triggers: none.
14. Evidence: booking ID, status and version; or block ID and source; conflict IDs.

**S3 — Check the new time and collect decisions**
1. Objective: confirm the new time is free and the busy data is fresh; for `busy_range`, get the owner's decision per conflict.
2. Capability: `nova.calendar.availability.get_slots.default`, `nova.calendar.availability.list_calendar`, `nova.calendar.busy_sync.status`.
3. Input mapping: `new_start` (UTC), duration (from the booking), booking ID (excluded from its own conflict check).
4. Dependencies: S2.
5. Preconditions: action is `reschedule` or `busy_range` (skipped for cancel, block without conflicts, and unblock).
6. Action / side effects: read-only. For each `busy_range` conflict ask: reschedule (offer up to 3 free slots), cancel, or keep (the meeting stays; the block is still added around it).
7. Autonomy: A2 for reads; A0 for the per-conflict choice.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: new start free, or alternatives offered; sync freshness recorded; one decision per conflict.
11. Verification: `new_start` is in the returned slots for the duration, and an independent `list_calendar` read shows no overlap.
12. Failure routes: slot not free → offer up to 3 nearest free slots; sync older than twice its interval → disclose possibly stale availability and ask whether to proceed; no decision → `blocked`.
13. Escalation triggers: stale sync and the owner insists → proceed only with that acknowledgement recorded in the approval.
14. Evidence: slot excerpt, sync timestamps, per-conflict decisions.

**S4 — Draft the change summary**
1. Objective: an exact, owner-readable statement of what will change and who will be emailed.
2. Capability: deterministic time formatter plus LLM wording (planner).
3. Input mapping: booking, new start and end, owner and guest zones, `send_updates`, `reason`.
4. Dependencies: S3 (S2 for cancel).
5. Preconditions: target resolved; new time free.
6. Action / side effects: for example "Move 'Intro call' with John Roe from Thu 5 Nov 11:00 New York (16:00 London) to Fri 6 Nov 09:30 New York (14:30 London). Same link. John will be emailed." No side effects.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 1 retry.
9. Idempotency: not applicable.
10. Postconditions: summary with UTC and both local times, guest list (emails masked in shared channels), notification mode.
11. Verification: the formatter re-derives every time from UTC and the text matches; guests equal the booking's guests; no added facts.
12. Failure routes: mismatch → regenerate from the formatter output only → else `blocked`.
13. Escalation triggers: none.
14. Evidence: summary text and its hash.

**S5 — Approval checkpoint**
1. Objective: explicit approval for each guest-notifying change.
2. Capability: `nova.approvals.gateway.request`, then `nova.approvals.gateway.status`.
3. Input mapping: summary hash, booking ID, booking version, action, `new_start`, `send_updates`.
4. Dependencies: S4.
5. Preconditions: action is reschedule or cancel (one approval per `busy_range` conflict). Block and unblock on an explicit owner request skip S5.
6. Action / side effects: Approve / Edit / Deny sent to the owner only; nothing changes yet.
7. Autonomy: A0 — human decision.
8. Timeout / retry: wait up to 30 minutes, then `expired` → `cancelled` for that change.
9. Idempotency: the approval ID binds one change at one booking version.
10. Postconditions: an approval record with ID, scope hash and expiry.
11. Verification: the approval's scope hash equals the summary hash and its booking version equals the current version.
12. Failure routes: Edit → S3/S4; Deny or expiry → `cancelled`, nothing changed.
13. Escalation triggers: not applicable.
14. Evidence: approval ID, decision, timestamp, scope hash.

**S6 — Apply the booking change**
1. Objective: reschedule or cancel exactly once.
2. Capability: `nova.calendar.booking.reschedule.default` or `nova.calendar.booking.cancel.default`; the propagation point then invokes `nova.calendar.invites.push` (`send-all` or `send-none` per the approval).
3. Input mapping: booking ID, `new_start`, approval ID, idempotency key, expected version, `send_updates`.
4. Dependencies: S5.
5. Preconditions: approval valid and unexpired; booking version unchanged (a guest may have used the self-serve page); new slot still free (S3 re-run).
6. Action / side effects: reschedule — the booking keeps its ID and meeting link; the old slot becomes a `rescheduled` history row; the self-manage token is transferred to the live row; the external event is moved and the live row keeps the external event ID; guests emailed under `send-all`. Cancel — status `cancelled`; the external event ID is returned; the event is deleted; guests emailed under `send-all`.
7. Autonomy: A1, covered by the S5 approval.
8. Timeout / retry: 20 s; no blind retry — on timeout go to S8 and read state by booking ID.
9. Idempotency: reschedule key as in Section F; cancel of a cancelled booking returns `already_in_state` and `not_repeated`.
10. Postconditions: one transaction committed; propagation queued exactly once.
11. Verification: returned booking ID equals the original; meeting link unchanged; external event ID still present when one existed.
12. Failure routes: version conflict → stop, re-read, new approval; slot taken → `blocked` with alternatives; database error → transaction rolled back, nothing changed → `failed`.
13. Escalation triggers: repeated errors; an orphaned external event (booking has none but the provider has one).
14. Evidence: operation result, idempotency key, propagation request reference.

**S7 — Apply the block or unblock**
1. Objective: add or remove an owner-only busy block.
2. Capability: `nova.calendar.blocks.block` or `nova.calendar.blocks.unblock`.
3. Input mapping: `range_start`, `range_end` (UTC), `title`, client key; or `block_id`.
4. Dependencies: S2; for `busy_range`, after every approved conflict change is applied or declined.
5. Preconditions: explicit owner request; `range_start < range_end`; for unblock, `source = manual`.
6. Action / side effects: create or delete the block; nobody else is notified; slots in the range stop or start being offered.
7. Autonomy: A2 on the explicit owner request; no approval.
8. Timeout / retry: 10 s / 2 retries (idempotent).
9. Idempotency: block client key returns the same block on replay; unblock of an absent block returns `already_in_state`.
10. Postconditions: block exists with exact bounds, or is gone.
11. Verification: `list_calendar` shows the exact bounds; `get_slots` offers no start whose span overlaps the range; a slot starting exactly at `range_end` is still offered.
12. Failure routes: empty range → reject; synced block → refuse; error → `failed`.
13. Escalation triggers: none.
14. Evidence: block ID, bounds, slot check excerpt.

**S8 — Verify state and propagation**
1. Objective: prove the intended outcome, not just a successful return.
2. Capability: `nova.calendar.booking.list`, `nova.calendar.availability.list_calendar`.
3. Input mapping: booking or block IDs, idempotency keys.
4. Dependencies: S6 and/or S7.
5. Preconditions: an apply step returned or timed out.
6. Action / side effects: read-only; read status, times, link, external event ID and propagation state.
7. Autonomy: A2.
8. Timeout / retry: 30 s; up to 3 polls 10 s apart for propagation.
9. Idempotency: read-only.
10. Postconditions: state equals the approved change; propagation `sent`, `not_applicable` (no external event) or `failed`.
11. Verification: exactly one live row per booking ID; one history row per reschedule; the old time is free and the new time is busy; at most one propagation request per change.
12. Failure routes: propagation failed or unknown → `partially_completed`, tell the owner guests may not have been emailed and offer a manual resend (new approval) — never resend automatically; state mismatch → `failed` and escalate.
13. Escalation triggers: propagation failure; duplicate live rows.
14. Evidence: post-change records; propagation status code.

**S9 — Deliver the result**
1. Objective: tell the owner exactly what changed.
2. Capability: `nova.notify.owner.send`.
3. Input mapping: `change_result`.
4. Dependencies: S8.
5. Preconditions: result assembled.
6. Action / side effects: plain-text message to the owner (no markup, so chat parsers cannot reject it).
7. Autonomy: A2.
8. Timeout / retry: 15 s / 2 retries.
9. Idempotency: message key = `run_id`.
10. Postconditions: delivered.
11. Verification: delivery receipt.
12. Failure routes: delivery fails → keep the result in the run record; mark `partially_completed`.
13. Escalation triggers: none.
14. Evidence: message ID.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Input validity | request matches `request@0.1.0`; `range_start < range_end` | validator | valid | ask once, then `blocked` |
| Time conversion | UTC re-formatted in both zones equals the summary text | formatter | exact | regenerate summary |
| Target identity | changed booking ID = approved booking ID | approval vs result | equal | stop, `failed`, report |
| Approval scope | scope hash and booking version match at apply time | approval record | equal | stop before change; re-approve |
| Slot freedom | new slot free in `get_slots` and `list_calendar` | two reads | both free | offer alternatives |
| Reschedule integrity | same ID and link; one history row; token on live row only | booking records | all hold | escalate |
| Block effect | no offered slot overlaps the block; adjacent slot still offered | `get_slots` | holds | `failed` |
| Synced-block guard | unblock targets `source = manual` only | block record | manual | refuse |
| Propagation | exactly one request; success code, or `not_applicable` | propagation log | holds | `partially_completed`, disclose |
| Output schema | `change_result` valid | validator | valid | fix fields; never fake values |
| Tenant scope | every object belongs to the owner's tenant | runtime | all | deny and escalate |

A successful `reschedule` or `cancel` return proves only that the database committed; guest notification is proven separately by the propagation status.

## J. Self-correction and recovery

- **Failure classes:** validation, ambiguous target, slot taken, stale busy data, permission denial, version conflict, timeout, transient provider error, propagation failure, failed assertion.
- **Permitted corrections:** ask once for missing input; retry reads (bounded); offer alternative slots; reconcile a timed-out change by reading state; re-request approval after a version conflict.
- **Forbidden corrections:** retrying a change on an unknown outcome; switching `send-all` to `send-none`; calling `invites.push` directly; removing a synced block; copying a self-manage token; weakening the past-time guard or the empty-range check.
- **Maximum correction cycles:** 1 per step; 3 per run.
- **Reverification required after correction:** yes — re-run S3 and the S5 scope check before any change.
- **Stop conditions:** correction budget exhausted; hard constraint fails; propagation failed (no further changes in a `busy_range` run).
- **Recovery or compensation:** an approved reschedule back to the old time; removal of a wrongly created block; a cancelled meeting can only be rebooked (new invite).
- **What must be reported:** every change applied, every email the provider sent, every correction, and any unresolved state.

## K. Independent challenge

- **Assumptions to challenge:** the matched booking is the one the owner meant; the owner's local time was read in the right zone; the guest's zone is current.
- **Alternative approach:** a short delay may be better as a message to the guest than a move; a block may be enough instead of a cancel.
- **Contradictory evidence:** the guest already moved or cancelled through the self-serve page; the busy-store is stale; a synced block overlaps the new slot.
- **Could mislead:** "Moved" while the provider never emailed the guest; "Blocked" with a zero-length block; a time shown in the wrong zone after a daylight-saving change.
- **Separate verifier:** S8 re-reads state independently of the apply step; the formatter, not the LLM, owns every displayed time.
- **Findings requiring disclosure:** propagation failure, stale availability, conflicts kept inside a block, history rows created.

## L. Human escalation

- **Triggers:** ambiguous booking; booking changed since approval; propagation failure; duplicate live rows; an orphaned external event; stale sync the owner wants to override.
- **Why the agent cannot decide:** guests are affected and emails cannot be recalled.
- **Information presented:** booking, old and new times in both zones, guests, approval state, propagation status.
- **Decision requested:** approve / edit / deny; resend the notification (new approval); accept the partial result.
- **Options and consequences:** a resend emails guests again; denying leaves the booking unchanged.
- **Safe behaviour while waiting:** no change in progress; nothing retried.
- **If no response:** after 30 minutes the pending change is `cancelled`; already-applied changes stay and are reported.
- **Resume:** a fresh approval against the current booking version; S3 re-runs first.

## M. Completion conditions

Mandatory: `change_result` valid; every applied change matches its approval (or the owner's explicit block request); S8 verification passed; propagation status known and reported; no synced block removed; evidence references present; limitations disclosed.

- `completed` — every requested change applied and verified, propagation `sent` or `not_applicable`, result delivered; also an idempotent no-op (already cancelled, already unblocked) reported as such.
- `partially_completed` — some `busy_range` changes applied and others declined or failed, or propagation failed or is unknown, or the result could not be delivered.
- `blocked` — validation, target not found, slot unavailable, synced-block refusal, missing capability or permission prevented any change.
- `failed` — a change was attempted but errored, or verification found an unintended state that could not be corrected.
- `cancelled` — the owner denied or the approval expired before any change.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome; `change_result`; step statuses S1–S9; verification results and failed checks; evidence references (approval IDs, booking and block IDs, propagation references); approvals and scope; side effects (changes applied, guests emailed) and reconciliation status; corrections performed; unresolved challenges; limitations; escalation state; learning candidate. Never include tokens, self-manage tokens or full guest emails in shared channels.

## O. Learning and revision

- **Outcome metrics:** approval rate, slot-taken rate, propagation success rate, version-conflict rate, idempotent-replay count, time from request to verified change.
- **Evidence retained:** run record, approval record and post-change booking state, per the retention policy.
- **Data minimisation:** metrics stored without guest names or emails.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}`.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review; no silent changes to trusted behaviour.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path — reschedule (database-only booking, no external event) | same ID and link at new time; one history row; propagation `not_applicable` | UNVERIFIED on Skill Port; observed in the reference implementation |
| Happy path — cancel and block (sandbox calendar, `send-none`) | status `cancelled`; block excludes slots; no email sent | UNVERIFIED on Skill Port; observed in the reference implementation |
| Missing input (no new time) | asks once, then `blocked` | TBD |
| Invalid input (zero-length or negative block) | rejected before any call | TBD |
| Invalid input (past new start; daylight-saving gap time) | rejected; asks for another time | TBD |
| Ambiguous booking | lists matches and asks | TBD |
| Missing capability (propagation binding absent) | `blocked` before reschedule or cancel; block still allowed | TBD |
| Permission denied (`calendar:bookings:write` missing) | `blocked`, nothing changed | TBD |
| Unauthorized scope (another tenant's booking ID) | denied | TBD |
| Tenant isolation | cannot list another tenant's bookings or blocks | TBD |
| Unblock a synced block | refused with "remove at source" | TBD |
| Adjacency | block ending at 10:00 leaves a 10:00 slot offered | TBD |
| Timeout on reschedule | state read by ID; no second change | TBD |
| Duplicate / replay (same idempotency key) | existing result returned; one propagation request | TBD |
| Cancel already cancelled | `already_in_state`; no second email | TBD |
| Version conflict (guest moved it via self-serve page after approval) | stop; new approval required | TBD |
| False success (commit OK, propagation failed) | `partially_completed`, disclosed | TBD |
| Failed verification (summary time mismatch) | regenerated from formatter | TBD |
| Correction budget exhausted | stops safely | TBD |
| Escalation (duplicate live rows) | escalates; no further change | TBD |
| Cancellation (owner denies at S5) | `cancelled`, nothing changed | TBD |
| Rollback / compensation (database error mid-reschedule) | transaction rolled back; token still on original row | TBD |
| Output schema and status | valid `change_result`, correct status | TBD |
| Compatibility / regression | passes after a slot-engine or provider version change | TBD |

All tests use rolled-back transactions, database-only bookings without an external event, `send-none`, sandbox calendars and fictional `@example.com` guests. Never test on a real calendar: every create and cancel there emails the organizer and the guests. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, schema registry and approval UI — TBD (owner: publisher, with Skill Port).
- Cross-area IDs (`nova.approvals.gateway`, `nova.notify.owner`, contacts, research, memory, mail) must be reconciled with the other recipes in the package (owner: publisher).
- How the runtime exposes propagation status to the agent — TBD; the reference implementation reads the database's outbound-request log.
- Recurring series and multi-host meetings are not supported.
- Approval timeout default (30 minutes) and retention period — TBD (owner: operator).
- Reference-implementation evidence is historical and must be re-verified on a running instance before relying on it.
- Provider behaviour (which attendees are emailed, wording) is controlled by the calendar provider, not the recipe.
