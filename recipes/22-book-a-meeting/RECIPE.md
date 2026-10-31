# Book a meeting

## A. Identity and ownership

- **Recipe ID:** `nova.calendar.book-a-meeting` (proposed until assigned by Skill Port)
- **Name:** Book a meeting
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** productivity
- **Tags:** calendar, booking, invite, video-link, time-zones, human-approval, sandbox-testing · Vedaxi task type: none of the seven supported today
- **Summary:** On the user's explicit request, the agent resolves guests from the user's contact directory, converts the requested time explicitly between time zones, refuses past or too-soon times, re-checks the slot, gets approval for the exact invite, then creates one booking with a video link and sends the calendar invite — and verifies that the booking, the external calendar event and the attendee list all match what was approved.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** A calendar database (master busy-store) with an atomic booking function; an external calendar provider (for example Google Calendar) that creates the event, attaches a calendar-native video link (for example Google Meet) and emails attendees; optionally a self-hosted video-room provider; a contact directory; an approval channel; an owner notification channel. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — booking with a calendar-native video link, multi-guest invites and readback have run end-to-end in Nova's reference implementation. Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.contacts.directory` | Capability | Find a guest's email from approved sources | medium | UNVERIFIED |
| `nova.contacts.directory.lookup` | Operation (read) | Look a name up in assistant memory, then the contact directory, then ask the user | medium | UNVERIFIED |
| `nova.calendar.availability` | Capability | Confirm the requested interval is free | low | UNVERIFIED |
| `nova.calendar.availability.get_slots` | Operation (read) | Propose alternatives when the time is taken (variant `default`) | low | UNVERIFIED |
| `nova.calendar.availability.list_calendar` | Operation (read) | Check the exact interval, with buffers, for conflicts | medium | UNVERIFIED |
| `nova.calendar.booking` | Capability | Create and read bookings in the master busy-store | high | UNVERIFIED |
| `nova.calendar.booking.create` | Operation (create) | Atomically claim the slot and create one booking (MCP tool `create_booking`) | high | UNVERIFIED |
| `nova.calendar.booking.list` | Operation (read) | Read a booking back by ID or idempotency key (MCP tools `list_bookings`, `upcoming_bookings`) | low | UNVERIFIED |
| `nova.calendar.booking.cancel` | Operation (update) | Compensation only: cancel a booking that never reached the external calendar | medium | UNVERIFIED |
| `nova.calendar.video` | Capability | Produce the meeting's video link | low | UNVERIFIED |
| `nova.calendar.video.create_link` | Operation (create) | Create a video link for the booking | low | UNVERIFIED |
| `nova.calendar.invites` | Capability | Mirror the booking to the external calendar and notify attendees | high | UNVERIFIED |
| `nova.calendar.invites.push` | Operation (create + external side effect) | Create the external event with attendees and notification mode | high | UNVERIFIED |
| `nova.calendar.invites.get` | Operation (read) | Read the external event back for verification | low | UNVERIFIED |
| `nova.approvals.gateway` | Capability | Ask the user to approve one specific action | low | UNVERIFIED |
| `nova.approvals.gateway.request` | Operation (create) | Send an approve / edit / cancel request bound to a content hash | low | UNVERIFIED |
| `nova.approvals.gateway.status` | Operation (read) | Read the decision | low | UNVERIFIED |
| `nova.notify.owner` | Capability | Deliver a message to the requesting user's own channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Send the confirmation to the user | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.contacts.directory.lookup.memory` → `.directory` → `.ask-user` — read-only first two, then one question to the user. An email typed by the user is accepted; an email inferred from a spoken or misheard name is never accepted (a speech-recognition typo once produced a wrong guest address in the reference implementation).
- `nova.calendar.availability.get_slots.default` — owner variant; used only to suggest alternatives.
- `nova.calendar.video.create_link.calendar-native` — the link is created by the calendar provider together with the event (for example a Google Meet link). Preferred when the calendar connection supports it.
- `nova.calendar.video.create_link.self-hosted-room` — an open-source video room link generated immediately (for example a Jitsi room on the operator's domain). Used when the user chooses it or the calendar provider has no native video. The link is final when issued.
- `nova.calendar.invites.push.send-all` — production: attendees are emailed by the calendar provider. Eligible only in `production` mode with a valid S6 approval.
- `nova.calendar.invites.push.send-none` — sandbox and tests: the event is created with notifications off, on a sandbox calendar, for fictional guests only. **No fallback in either direction:** a failed `send-all` never retries as `send-none` (guests would silently not be told), and `send-none` is never promoted to `send-all`.
- `nova.calendar.booking.create.default`, `.list.default`, `.cancel.default`, `invites.get.default`, the approval and notification operations — only the default variant is supported.

**Bindings:** `binding://nova/contacts/lookup`, `binding://nova/calendar/get-slots`, `binding://nova/calendar/calendar-range`, `binding://nova/calendar/create-booking`, `binding://nova/calendar/list-bookings`, `binding://nova/calendar/cancel-booking`, `binding://nova/calendar/video-link`, `binding://nova/calendar/invite-push`, `binding://nova/calendar/invite-get`, `binding://nova/approvals/request`, `binding://nova/approvals/status`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: security-definer database functions (`book`, `list`, `upcoming`, `cancel`) called through the owner's assistant and an MCP server on an n8n automation hub; a workflow creates the external calendar event with a native video link and attendee notifications, stores the external event ID on the booking and swaps in the final link.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Guest | `schema://nova/book-a-meeting/guest@0.1.0` | the requesting user's directory | read |
| Booking | `schema://nova/book-a-meeting/booking@0.1.0` | the requesting user's tenant | create, read; cancel (compensation only) |
| External calendar event | `schema://nova/book-a-meeting/external-event@0.1.0` | the user's connected calendar | create, read |
| Video link | string on the Booking | linked to one Booking | create |
| Approval | `schema://nova/book-a-meeting/approval@0.1.0` | the requesting user | create, read |
| Guest self-manage link | tokenised URL on the Booking | linked to one Booking | create (with the booking) |

**Authorization requirements (declared, never a grant):** `contacts:read`, `calendar:availability:read`, `calendar:busy:read_detail`, `calendar:booking:create`, `calendar:booking:read`, `calendar:booking:cancel` (compensation only), `calendar:external:write` (event create with notifications), `calendar:external:read`, `approvals:request`, `notify:owner`. Calendar-provider credentials live in the runtime as `secret_ref://calendar-connection`; the self-hosted video domain as configuration, not a secret.

**Policy / trust:** risk class `high` (third parties receive an email invite from the user's calendar). Requires the S6 approval in production mode, the past-time guard, directory-only guest emails and a guest cap. Tests run only in `sandbox` mode.

## C. Intent and applicability

- **User problem:** The user wants a meeting on the calendar with the right people, at the right time in everyone's time zone, with a working video link and invites sent, without doing the calendar admin.
- **Desired outcome:** Exactly one booking and one external event, matching the approved time, guests and link, with invites delivered (production) or deliberately suppressed (sandbox).
- **Trigger or entry conditions:** The user explicitly asks to book, schedule or set up a meeting with named people (often after "Find free slots", recipe 21).
- **Applicable situations:** One-to-one or small-group meetings hosted by the user; online (video), in-person or phone location.
- **Out-of-scope situations:** A guest booking themselves through a public booking page (the guest supplies their own email under a different trust model); rescheduling or cancelling (recipe 23); recurring series; multi-host or round-robin; bulk or marketing invitations; booking on someone else's calendar.
- **Required inputs:** `guests`, `start`, `duration_min` or `event_type`, `title` (see Section D).
- **Prerequisites:** Working hours and owner time zone set; a connected external calendar with write access; a contact directory; an approval channel.
- **Required capabilities:** `nova.contacts.directory`, `nova.calendar.availability`, `nova.calendar.booking`, `nova.calendar.video`, `nova.calendar.invites`, `nova.approvals.gateway`, `nova.notify.owner` (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port; every binding is a dependency to resolve before activation.
- **Assumptions:** Verified in the reference implementation: atomic booking against the slot engine, calendar-native video links, multiple attendees per event, readback of the external event ID. Assumed: the target calendar provider supports a "no notifications" mode for sandbox tests.
- **Ambiguity policy:** Ask. If a guest, the time zone, the date or the length is unclear, ask one clarifying question; never guess an email or a time.
- **Vedaxi task type:** none of the seven supported today.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `guests` | array of 1–10 names or emails | yes | user; personal data | — | Name not found: ask for the email (typed). Two matches: ask which. Email not in directory and not typed by the user: reject. |
| `start` | local date-time + IANA zone, or a slot from recipe 21 (UTC) | yes | user | — | No zone: use the owner's zone and say so. Nonexistent local time (spring-forward gap): reject and ask. Ambiguous local time (fall-back hour): ask which. |
| `duration_min` / `event_type` | integer 15–240 / event-type slug | one of them | user | — | Missing: ask. Unknown slug: list valid slugs. |
| `title` | string 3–120 chars | yes | user | "Meeting: {owner} / {first guest}" | Contains an email or phone number: ask to remove. |
| `agenda` | string ≤ 1,000 chars | no | user | none | Over length: ask to shorten. |
| `location_type` | enum `online`, `in_person`, `phone` | no | user | `online` | `in_person` without an address: ask. |
| `video_variant` | enum `calendar-native`, `self-hosted-room` | no | user | `calendar-native` if supported | Not supported by the connection: tell the user; never switch silently. |
| `mode` | enum `production`, `sandbox` | no | runtime / tester | `production` | `sandbox` requires every guest on `example.com` / `example.org` or a tester-owned test domain; otherwise reject. |
| `include_manage_link` | boolean | no | user | `true` | — |

```json
{"$id":"schema://nova/book-a-meeting/request@0.1.0","type":"object","required":["guests","start","title"],
 "oneOf":[{"required":["duration_min"]},{"required":["event_type"]}],
 "properties":{"guests":{"type":"array","minItems":1,"maxItems":10,"items":{"type":"string","minLength":2}},
 "start":{"type":"object","required":["local"],"properties":{"local":{"type":"string","pattern":"^\\d{4}-\\d{2}-\\d{2}T\\d{2}:\\d{2}$"},
  "tz":{"type":"string"},"utc":{"type":"string","format":"date-time"}}},
 "duration_min":{"type":"integer","minimum":15,"maximum":240,"multipleOf":5},"event_type":{"type":"string"},
 "title":{"type":"string","minLength":3,"maxLength":120},"agenda":{"type":"string","maxLength":1000},
 "location_type":{"enum":["online","in_person","phone"]},"location_detail":{"type":"string","maxLength":200},
 "video_variant":{"enum":["calendar-native","self-hosted-room"]},"mode":{"enum":["production","sandbox"]},
 "include_manage_link":{"type":"boolean"}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `booking_result` | object `schema://nova/book-a-meeting/booking-result@0.1.0` | `status`, `booking_id`, `start_utc`, `end_utc`, `owner_tz`, `guests`, `notifications` | the requesting user's channel and the run record | personal data; owner only | readback of the booking and the external event (Section I) |

```json
{"$id":"schema://nova/book-a-meeting/booking-result@0.1.0","type":"object",
 "required":["status","booking_id","start_utc","end_utc","owner_tz","guests","notifications"],
 "properties":{"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},
 "booking_id":{"type":["string","null"]},"start_utc":{"type":"string","format":"date-time"},"end_utc":{"type":"string","format":"date-time"},
 "owner_tz":{"type":"string"},"start_owner_local":{"type":"string"},
 "guests":{"type":"array","items":{"type":"object","required":["name","email"],"properties":{"name":{"type":"string"},
  "email":{"type":"string","format":"email"},"tz":{"type":["string","null"]},"start_local":{"type":["string","null"]}}}},
 "video_link":{"type":["string","null"],"format":"uri"},"external_event_ref":{"type":["string","null"]},
 "notifications":{"enum":["send-all","send-none","not_sent"]},"manage_link_included":{"type":"boolean"},
 "approval_id":{"type":["string","null"]},"notes":{"type":"array","items":{"type":"string"}}}}
```

Other schemas:

```json
{"$id":"schema://nova/book-a-meeting/guest@0.1.0","type":"object","required":["name","email","source"],
 "properties":{"name":{"type":"string"},"email":{"type":"string","format":"email"},"tz":{"type":["string","null"]},
 "source":{"enum":["memory","directory","user_typed"]}}}
{"$id":"schema://nova/book-a-meeting/booking@0.1.0","type":"object","required":["booking_id","status","start_utc","end_utc","guest_emails"],
 "properties":{"booking_id":{"type":"string"},"status":{"enum":["confirmed","cancelled","rescheduled"]},
 "start_utc":{"type":"string","format":"date-time"},"end_utc":{"type":"string","format":"date-time"},
 "guest_emails":{"type":"array","items":{"type":"string","format":"email"}},"title":{"type":"string"},
 "location_type":{"enum":["online","in_person","phone"]},"video_link":{"type":["string","null"]},
 "video_link_state":{"enum":["none","provisional","final"]},"external_event_ref":{"type":["string","null"]},
 "idempotency_key":{"type":"string"},"test":{"type":"boolean"}}}
{"$id":"schema://nova/book-a-meeting/external-event@0.1.0","type":"object","required":["external_event_ref","start_utc","end_utc","attendees"],
 "properties":{"external_event_ref":{"type":"string"},"start_utc":{"type":"string","format":"date-time"},"end_utc":{"type":"string","format":"date-time"},
 "attendees":{"type":"array","items":{"type":"string","format":"email"}},"conference_link":{"type":["string","null"]},
 "notifications":{"enum":["send-all","send-none"]},"booking_key":{"type":"string"}}}
{"$id":"schema://nova/book-a-meeting/approval@0.1.0","type":"object","required":["approval_id","content_hash","decision"],
 "properties":{"approval_id":{"type":"string"},"content_hash":{"type":"string"},"decision":{"enum":["pending","approved","edit","cancelled","expired"]},
 "decided_at":{"type":["string","null"],"format":"date-time"},"expires_at":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/book-a-meeting/contact-query@0.1.0","type":"object","required":["query"],"properties":{"query":{"type":"string","minLength":2},"typed_email":{"type":"string","format":"email"}}}
{"$id":"schema://nova/book-a-meeting/range@0.1.0","type":"object","required":["from_utc","to_utc"],"properties":{"from_utc":{"type":"string","format":"date-time"},"to_utc":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/book-a-meeting/slot-query@0.1.0","type":"object","required":["duration_min","from","to"],"properties":{"duration_min":{"type":"integer","minimum":15,"maximum":240},"from":{"type":"string"},"to":{"type":"string"},"max_results":{"type":"integer","maximum":3}}}
{"$id":"schema://nova/book-a-meeting/conflict-result@0.1.0","type":"object","required":["free"],"properties":{"free":{"type":"boolean"},"conflicts":{"type":"array","items":{"type":"object","properties":{"start_utc":{"type":"string","format":"date-time"},"end_utc":{"type":"string","format":"date-time"}}}},"alternatives":{"type":"array","maxItems":3,"items":{"type":"object","properties":{"start_utc":{"type":"string","format":"date-time"},"end_utc":{"type":"string","format":"date-time"}}}}}}
{"$id":"schema://nova/book-a-meeting/booking-input@0.1.0","type":"object","required":["start_utc","end_utc","guest_emails","title","idempotency_key"],"properties":{"start_utc":{"type":"string","format":"date-time"},"end_utc":{"type":"string","format":"date-time"},"guest_emails":{"type":"array","minItems":1,"maxItems":10,"items":{"type":"string","format":"email"}},"title":{"type":"string"},"location_type":{"enum":["online","in_person","phone"]},"idempotency_key":{"type":"string"},"test":{"type":"boolean"}}}
{"$id":"schema://nova/book-a-meeting/booking-ref@0.1.0","type":"object","properties":{"booking_id":{"type":"string"},"idempotency_key":{"type":"string"},"external_event_ref":{"type":"string"}},"minProperties":1}
{"$id":"schema://nova/book-a-meeting/approval-request@0.1.0","type":"object","required":["summary","content_hash","expires_in_min"],"properties":{"summary":{"type":"string","maxLength":1200},"content_hash":{"type":"string"},"expires_in_min":{"type":"integer","maximum":60}}}
{"$id":"schema://nova/book-a-meeting/approval-ref@0.1.0","type":"object","required":["approval_id"],"properties":{"approval_id":{"type":"string"}}}
{"$id":"schema://nova/book-a-meeting/delivery@0.1.0","type":"object","required":["message_id","delivered"],"properties":{"message_id":{"type":"string"},"delivered":{"type":"boolean"}}}
```

**Example (fictional)** — owner Jane Doe (`America/New_York`) books John Roe (`Europe/London`, in the directory as `john@example.org`).

Input: `{"guests":["John Roe"],"start":{"local":"2026-11-03T14:00","tz":"Europe/London"},"duration_min":30,"title":"Project kickoff","location_type":"online","mode":"sandbox"}`

Output: `{"status":"completed","booking_id":"bk_0001","start_utc":"2026-11-03T14:00:00Z","end_utc":"2026-11-03T14:30:00Z","owner_tz":"America/New_York","start_owner_local":"Tue 3 Nov 09:00 EST","guests":[{"name":"John Roe","email":"john@example.org","tz":"Europe/London","start_local":"Tue 3 Nov 14:00 GMT"}],"video_link":"https://meet.example.com/abc-defg-hij","external_event_ref":"evt_sandbox_0001","notifications":"send-none","manage_link_included":true,"approval_id":"apr_0001","notes":["Sandbox mode: no invitation emails were sent."]}`

## E. Constraints and budgets

- **Hard constraints:** explicit user request; in `production` mode, the S6 approval covers this exact content hash (guests, start, end, title, location, notification mode); guest emails come only from assistant memory, the contact directory, or an email the user typed and confirmed — never derived from a spoken or misheard name and never invented; every time is converted explicitly from its stated zone to UTC through the IANA database and shown back in the owner's and each guest's zone; **past-time guard:** `start_utc ≥ now + 5 minutes` (a hard floor even for the owner), plus the policy minimum notice for non-owner contexts, and `start_utc ≤ now + horizon`; the slot is re-checked free in the same transaction that creates the booking; one booking and one external event per approval; at most 10 guests; the owner is the only host; a provisional or placeholder video link is never shown to the user or guests; **tests never touch a real calendar or real guests** — `sandbox` mode uses `send-none`, a sandbox calendar and fictional guests, and a test booking with a real person's email is never persisted anywhere a background job could pick it up (a persisted synthetic booking once caused a real person to receive an invite in the reference implementation).
- **Soft constraints:** prefer times inside both parties' working hours; keep titles short; include the guest self-manage link.
- **Resource and tenant scope:** the requesting user's directory, busy-store and connected calendar only.
- **Data sources allowed:** assistant memory, the contact directory, the busy-store, the user's own messages.
- **External transfers permitted:** the event (title, time, attendees, agenda, link) to the user's calendar provider, which emails attendees in `production` mode.
- **Time limit:** whole run ≤ 10 minutes excluding the wait for approval; approval wait ≤ 60 minutes.
- **Cost or token budget:** calendar API calls ≤ 6 per run; optional LLM drafting of the agenda within an operator cap (`TBD`).
- **Maximum steps:** 10 (S1–S10).
- **Maximum retries and correction cycles:** reads 2 retries; `booking.create` 1 retry only after a readback by idempotency key; `invites.push` 1 retry only after an `invites.get` readback; correction cycles 1 per step, 3 per run.
- **Rate limits and concurrency:** one booking run per user at a time; calendar-provider quotas apply.
- **Retention and logging constraints:** the run record keeps booking ID, times, guest count and approval ID; logs never contain the full guest list in shared channels or any credential.
- **Prohibited actions:** booking without approval in production; adding guests not in the approved list; changing the time after approval without a new approval; booking on another person's calendar; sending invites from a sandbox run.
- **Stop conditions:** approval declined or expired; past-time guard fails; slot taken and the user declines the alternatives; guest unresolved; calendar connection lacks write access.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Booking (one per approval) | S1–S6 passed; slot free at commit | Approval Required (S6) in production; Allowed in sandbox | readback by ID: times, guests, status `confirmed` |
| Create | External calendar event with attendees and video link | booking exists | Approval Required (same S6 approval) | `invites.get`: attendees, times and link match |
| Create | Approval record | draft ready | Allowed | decision readable |
| Read | Guests, busy intervals, bookings, external event | own tenant only | Allowed | — |
| Update | Booking `video_link`, `video_link_state`, `external_event_ref` | set once after the push | Allowed | readback shows `final` link and event ref |
| Update | Booking `status` → `cancelled` | compensation only: the push failed and no external event exists | Approval Required (the user chooses this option) | status `cancelled`; no event exists, so no email is sent |
| Delete | any | — | Not Allowed | — |
| External side effect | invitation emails to guests (`send-all`) | production mode; approval valid | Approval Required (S6) | event attendees list equals approved guests |
| External side effect | confirmation message to the user | run finished | Allowed | delivery receipt |

- **Idempotency:** booking key = hash(`owner_id`, `start_utc`, `end_utc`, sorted guest emails, `approval_id`); a second `create` with the same key returns the existing booking. The external event carries the booking key as a private property; a second `push` for a booking that already has an `external_event_ref` updates instead of creating, so no second invite is sent.
- **Expected side effects:** a busy interval appears in the busy-store; an event appears on the user's calendar; in production, every guest receives an invitation email from the calendar provider.
- **Compensation:** if the push failed and no external event exists, the booking can be cancelled with no one notified. Once invites are sent, any correction is a reschedule or cancellation through recipe 23, which notifies guests and needs its own approval.
- **Irreversible:** invitation emails, once sent.
- **Ambiguous outcomes:** if `create` times out, read bookings by idempotency key before any retry; if `push` times out, call `invites.get` by booking key before any retry; never push again on an unknown outcome.

## G. Autonomy and approval policy

- **Default autonomy level:** A1 — supervised execution.
- **Planning level:** A2 (resolve guests, convert times, check conflicts, draft the invite).
- **Execution level:** A1 — create and push only after the S6 approval (production); A2 in sandbox mode, which cannot notify anyone.
- **Verification level:** A2.
- **Recovery level:** A1 — no second push or new time without a new approval.
- **Approval required for:** the booking plus invite (exact guests, time in each zone, title, location, link type, notification mode); cancelling a booking as compensation.
- **Actions never permitted:** sending invites without approval; inventing or inferring an email; booking in the past; adding unapproved guests; testing in production mode; overwriting another booking.
- **Conditions requiring clarification:** unknown or ambiguous guest; missing zone where the owner's default would be surprising (for example a guest-stated time); nonexistent or ambiguous local time; missing length.
- **Conditions requiring escalation:** calendar write access missing; push fails twice; readback mismatch after invites were sent; more than 10 guests requested.
- **Safe state after stop, cancellation or escalation:** no booking created, or a booking with no external event (cancelled if the user chooses), nothing re-pushed, the user told exactly what exists.

Approval covers one booking with the stated content. It does not cover later edits, other guests or other times.

## H. Execution plan

**S1 — Validate the request**
1. Objective: guests, start, length and title present and valid.
2. Capability: none (local validation).
3. Input mapping: user message → `request@0.1.0`.
4. Dependencies: none.
5. Preconditions: explicit request to book.
6. Action / side effects: parse and validate; resolve `event_type` to minutes later in S4; no side effects.
7. Autonomy: A2; no approval.
8. Timeout / retry: 5 s / none.
9. Idempotency: not applicable.
10. Postconditions: schema-valid request; `mode` fixed.
11. Verification: JSON Schema passes; in `sandbox` mode every typed guest email is on a fictional or tester-owned domain.
12. Failure routes: missing or invalid field → ask once → still missing → `blocked`.
13. Escalation triggers: more than 10 guests.
14. Evidence: validated request.

**S2 — Resolve guests from the directory**
1. Objective: exactly one email per guest from approved sources.
2. Capability: `nova.contacts.directory.lookup` (variants memory → directory → ask-user).
3. Input mapping: each name → `query`.
4. Dependencies: S1.
5. Preconditions: guest given as a name (skip lookup if the user typed the email).
6. Action / side effects: read-only lookups; when the match is fuzzy (for example a name heard by speech recognition), show the candidate name with a masked email and ask the user to confirm.
7. Autonomy: A2 for reads; A0 for confirmations.
8. Timeout / retry: 10 s per variant / 2 retries.
9. Idempotency: read-only.
10. Postconditions: one `guest@0.1.0` per guest with `source`.
11. Verification: every email came from memory, the directory or a user-typed entry; none built from a name pattern.
12. Failure routes: multiple matches → ask; none → ask-user; user cannot supply → `blocked`.
13. Escalation triggers: none.
14. Evidence: source per guest (not the full directory).

**S3 — Convert time zones and apply the past-time guard**
1. Objective: an unambiguous UTC interval that is in the future and inside the horizon.
2. Capability: none (IANA conversion in the runtime); policy from `nova.calendar.availability.get_slots` metadata.
3. Input mapping: `start.local` + `start.tz` (or the owner's zone, stated back to the user), duration.
4. Dependencies: S1.
5. Preconditions: zone is a valid IANA name.
6. Action / side effects: convert to UTC; detect nonexistent and ambiguous local times; compute `end_utc`; render the time in the owner's zone and each guest's known zone.
7. Autonomy: A2.
8. Timeout / retry: 2 s / none.
9. Idempotency: not applicable.
10. Postconditions: `start_utc`, `end_utc`, renderings per party.
11. Verification: round trip local → UTC → local returns the input; `start_utc ≥ now + 5 min` (and policy notice for non-owner contexts); `start_utc ≤ now + horizon`.
12. Failure routes: in the past or inside the floor → `blocked` with "that time has passed — next free slots are…" (offer recipe 21 results); nonexistent or ambiguous local time → ask.
13. Escalation triggers: none.
14. Evidence: input time, zone used, UTC result, guard results.

**S4 — Check for conflicts**
1. Objective: confirm the interval is free, including buffers.
2. Capability: `nova.calendar.availability.list_calendar`; `get_slots.default` for alternatives.
3. Input mapping: `start_utc` − buffer → `end_utc` + buffer.
4. Dependencies: S3.
5. Preconditions: valid interval.
6. Action / side effects: read-only; half-open overlap test against bookings (with buffers) and busy blocks; check working hours and the daily cap.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: free, or a conflict with up to 3 alternatives.
11. Verification: no overlapping interval in the range.
12. Failure routes: conflict → show the 3 nearest free slots in the guest's zone and ask; outside working hours → ask the owner to confirm explicitly.
13. Escalation triggers: none.
14. Evidence: conflict check result (times only).

**S5 — Draft the invite**
1. Objective: the exact content guests will receive.
2. Capability: the planner (optional mid-tier LLM for agenda wording only).
3. Input mapping: title, agenda, guests, renderings, location, `video_variant`, `include_manage_link`, `mode`.
4. Dependencies: S2, S4.
5. Preconditions: guests resolved; interval free.
6. Action / side effects: build the invite text: title, time in each party's zone, agenda, location, "video link will be attached", self-manage link note, notification mode; no side effects.
7. Autonomy: A2.
8. Timeout / retry: 20 s / 1 retry.
9. Idempotency: not applicable.
10. Postconditions: draft plus content hash over guests, `start_utc`, `end_utc`, title, location and notification mode.
11. Verification: the draft adds no facts beyond the request; title and agenda contain no phone numbers or other guests' details.
12. Failure routes: verification fails → redraft once → else `blocked`.
13. Escalation triggers: none.
14. Evidence: draft and hash.

**S6 — Approval checkpoint**
1. Objective: explicit approval of this exact booking and invite.
2. Capability: `nova.approvals.gateway.request` and `.status` (production); in sandbox, a tester confirmation is recorded instead.
3. Input mapping: draft, masked guest emails in shared channels, times in each zone, notification mode.
4. Dependencies: S5.
5. Preconditions: draft verified.
6. Action / side effects: ask "Book 'Project kickoff' with John Roe on Tue 3 Nov, 09:00 New York / 14:00 London, video link, invite emailed?" with Approve / Edit / Cancel.
7. Autonomy: A0 — human decision.
8. Timeout / retry: wait up to 60 minutes, then `cancelled`.
9. Idempotency: the approval ID binds one content hash.
10. Postconditions: approval record with ID, hash and time.
11. Verification: approved hash equals the draft hash.
12. Failure routes: Edit → back to S3 or S5 as needed; Cancel or expiry → `cancelled`.
13. Escalation triggers: not applicable.
14. Evidence: approval ID, timestamp, hash.

**S7 — Create the booking**
1. Objective: one booking that claims the slot atomically.
2. Capability: `nova.calendar.booking.create.default`.
3. Input mapping: `start_utc`, `end_utc`, guest emails, title, location, idempotency key, `test` = (`mode` = `sandbox`).
4. Dependencies: S6.
5. Preconditions: approval valid and unexpired; past-time guard re-checked (time may have passed while waiting).
6. Action / side effects: the database re-tests the slot and inserts the booking in one transaction; a busy interval now exists.
7. Autonomy: A1, covered by the S6 approval.
8. Timeout / retry: 15 s / 1 retry only after a readback by key shows no booking.
9. Idempotency: key = hash(`owner_id`, `start_utc`, `end_utc`, sorted guest emails, `approval_id`).
10. Postconditions: booking `confirmed` with `booking_id`.
11. Verification: readback by ID shows the approved times and guests.
12. Failure routes: slot taken at commit → `blocked`, offer alternatives (new approval needed); timeout → readback by key.
13. Escalation triggers: none.
14. Evidence: booking ID and key.

**S8 — Attach the video link and push the invite**
1. Objective: one external event with the approved attendees and a final video link.
2. Capability: `nova.calendar.video.create_link` (selected variant) and `nova.calendar.invites.push` (`send-all` in production, `send-none` in sandbox).
3. Input mapping: booking, guests, notification variant, booking key.
4. Dependencies: S7.
5. Preconditions: booking `confirmed`; no `external_event_ref` yet (else update path).
6. Action / side effects: create the external event with attendees (and, for `calendar-native`, a conference request); in production the provider emails each guest; write the final link and the event ref onto the booking. If the architecture issues a provisional link first, wait up to 30 s for the final link and never surface the provisional one.
7. Autonomy: A1, covered by the S6 approval.
8. Timeout / retry: 30 s / 1 retry only after `invites.get` by booking key shows no event.
9. Idempotency: the booking key on the event; an existing `external_event_ref` turns a retry into an update.
10. Postconditions: booking has `video_link_state = final` and `external_event_ref`.
11. Verification: the returned event's attendee set equals the approved guest set.
12. Failure routes: push fails twice → booking kept without invites; ask the user: retry later, or cancel the booking (nobody is notified because no event exists); final link not ready in 30 s → report "link pending" honestly and re-check in S9.
13. Escalation triggers: two push failures; calendar write access missing.
14. Evidence: event ref, notification mode, link state.

**S9 — Verify booking and invite**
1. Objective: prove the booking, the event and the approval agree.
2. Capability: `nova.calendar.booking.list` and `nova.calendar.invites.get`.
3. Input mapping: `booking_id`, `external_event_ref`.
4. Dependencies: S8.
5. Preconditions: event created.
6. Action / side effects: read-only.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: verification record.
11. Verification: booking and event times equal the approved `start_utc` / `end_utc`; attendees equal the approved guests (no more, no fewer); event link equals the booking link; notification mode equals the approved mode; exactly one event carries the booking key.
12. Failure routes: mismatch before guests were notified (sandbox) → fix and re-verify; mismatch after invites were sent → escalate (correction is recipe 23 with a new approval).
13. Escalation triggers: any post-send mismatch; a second event with the same booking key.
14. Evidence: field-by-field comparison.

**S10 — Deliver the confirmation**
1. Objective: tell the user what exists.
2. Capability: `nova.notify.owner.send`.
3. Input mapping: `booking_result`.
4. Dependencies: S9.
5. Preconditions: result assembled.
6. Action / side effects: send the confirmation (times in each zone, guests, link, whether invites were emailed).
7. Autonomy: A1.
8. Timeout / retry: 15 s / 2 retries.
9. Idempotency: message key = `run_id`.
10. Postconditions: delivered.
11. Verification: delivery receipt.
12. Failure routes: delivery fails → keep the result in the run record; `partially_completed`.
13. Escalation triggers: none.
14. Evidence: message ID.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Input validity | request matches `request@0.1.0` | validator | valid | ask once, then `blocked` |
| Guest provenance | each email from memory, directory or user-typed | S2 record | 100 % | stop before S5 |
| Time conversion | local → UTC → local round trip equals input | S3 check | equal | ask the user |
| Past-time guard | `start_utc ≥ now + 5 min` at S3 and again at S7 | clock vs request | pass | `blocked`, offer slots |
| Conflict | interval free with buffers at S4 and at commit (S7) | busy-store | free | offer alternatives |
| Approval scope | approved hash = draft hash = created content | approval vs booking vs event | equal | stop; escalate if already sent |
| Attendee identity | event attendees = approved guests | `invites.get` | exact set match | escalate |
| Link truth | link shown is `final` | booking `video_link_state` | `final` | report "pending" |
| Mode integrity | sandbox ⇒ `send-none` and fictional guests | event and request | consistent | stop; defect |
| Output schema | `booking_result` valid | validator | valid | fix; never fake values |

The provider accepting an event proves only that a request was made; S9 shows the right people were invited at the right time.

## J. Self-correction and recovery

- **Failure classes:** validation, unresolved guest, past or nonexistent time, slot conflict, permission denial, timeout, transient provider error, ambiguous create or push, readback mismatch.
- **Permitted corrections:** ask once for missing input; offer alternative slots (needs re-approval); retry reads; retry `create` or `push` once after a readback shows nothing exists; redraft once.
- **Forbidden corrections:** guessing an email; moving the time without approval; switching `send-all` to `send-none` (or back); pushing again on an unknown outcome; adding or dropping guests silently; cancelling a notified booking without recipe 23's approval.
- **Maximum correction cycles:** 1 per step; 3 per run.
- **Reverification required after correction:** yes — S3 guard and S4 conflict check re-run before S7; S9 re-runs after any S8 change.
- **Stop conditions:** correction budget exhausted; hard constraint fails.
- **Recovery or compensation:** a booking with no external event may be cancelled at the user's choice (nobody notified); anything already sent is corrected through recipe 23.
- **What must be reported:** every correction, whether invites were sent, any booking left without an event, any mismatch.

## K. Independent challenge

- **Assumptions to challenge:** that "3 pm" meant the owner's zone rather than the guest's; that the matched contact is the intended person; that the guest's zone in the directory is current.
- **Alternative approach:** if the guest's working hours are unknown, would sending two or three options (recipe 21) be kinder than a fixed invite? Offer it.
- **Contradictory evidence:** the guest's directory zone differs from the zone in the user's message; an existing booking with the same guest the same day.
- **Could mislead:** a confirmation that shows only the owner's zone; a "booked" message when the push failed; a placeholder link.
- **Separate verifier:** S9 compares the stored booking, the external event and the approval hash in code separate from the steps that created them.
- **Findings requiring disclosure:** sandbox mode (no emails), link pending, a guest added from a user-typed email, any zone assumption.

## L. Human escalation

- **Triggers:** ambiguous guest; time outside working hours; calendar write access missing; two push failures; post-send mismatch; more than 10 guests.
- **Why the agent cannot decide:** identity of the guest, the owner's time, and anything already emailed to third parties.
- **Information presented:** guests (masked), times in each zone, conflict results, booking and event state, what was sent.
- **Decision requested:** approve / edit / cancel; reconnect the calendar; keep or cancel an un-invited booking; correct via recipe 23.
- **Options and consequences:** cancelling after invites sends cancellation emails; keeping a mismatched event confuses guests.
- **Safe behaviour while waiting:** nothing re-pushed; no new booking.
- **If no response:** after 60 minutes the run ends `cancelled` (before S7) or `partially_completed` (after S7) with evidence saved.
- **Resume:** a new approval is required; S3 and S4 re-run first.

## M. Completion conditions

Mandatory: `booking_result` valid; guest emails from approved sources; past-time guard passed at S3 and S7; one booking and one event matching the approval; link `final`; notification mode as approved; limitations disclosed; evidence references present.

- `completed` — booking and event verified, link final, invites sent (production) or deliberately suppressed (sandbox), confirmation delivered.
- `partially_completed` — booking exists but the invite push failed, the link is still pending, or the confirmation could not be delivered.
- `blocked` — input, guest resolution, past-time guard, conflict, permission or approval prevented the booking.
- `failed` — creation or push failed beyond the correction budget, or verification found a mismatch that could not be corrected.
- `cancelled` — the user declined or the approval expired.

A booking whose invites failed is never reported as `completed`.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome; `booking_result`; step statuses S1–S10; verification results and failed checks; evidence references (approval ID, booking ID, external event ref, conversion record); approvals and scope; side effects (booking created, invites sent or suppressed) and reconciliation status; corrections performed; unresolved challenges; limitations; escalation state; learning candidate. Never include credentials, the full guest list in shared channels, or the self-manage token.

## O. Learning and revision

- **Outcome metrics:** bookings per request, conflict rate at S4 and at commit, approval edit rate, push failure rate, readback mismatch rate, past-time guard hits, wrong-guest corrections reported later.
- **Evidence retained:** run record, approval record, booking and event references, per the retention policy.
- **Data minimisation:** metrics without guest emails or titles.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}`.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review; no silent changes to trusted behaviour.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (sandbox mode, sandbox calendar, fictional guests, `send-none`) | one booking, one event, final link, verified; no email sent | UNVERIFIED on Skill Port; observed in the reference implementation |
| Missing title or length | asks once, then `blocked` | TBD |
| Invalid zone / nonexistent local time | asked or rejected | TBD |
| Past-time guard (start 10 minutes ago; start in 2 minutes) | `blocked`, alternatives offered | TBD |
| Guest not in directory | asks for a typed email; never infers one | TBD |
| Misheard guest name | fuzzy candidate shown and confirmed | TBD |
| Missing capability (invite binding absent) | `blocked` before S7 | TBD |
| Permission denied (`calendar:external:write` missing) | `blocked`, no booking | TBD |
| Unauthorized scope (another tenant's calendar) | denied | TBD |
| Tenant isolation | cannot read another tenant's bookings or contacts | TBD |
| Slot taken between S4 and S7 | commit refused; alternatives need re-approval | TBD |
| Timeout on `create` | readback by key; no duplicate booking | TBD |
| Timeout on `push` | `invites.get` readback; no second invite | TBD |
| Duplicate / replay (same idempotency key) | existing booking returned | TBD |
| False success (provider accepted, attendee list wrong) | S9 detects; escalated | TBD |
| Failed verification (sandbox readback time differs from approval) | corrected and re-verified once, else `failed` | TBD |
| Placeholder link | never shown; "pending" reported | TBD |
| Sandbox with a real-looking guest domain | rejected at S1 | TBD |
| Correction budget exhausted | stops safely | TBD |
| Escalation (calendar disconnected) | user asked to reconnect | TBD |
| Cancellation (user declines at S6) | `cancelled`, nothing created | TBD |
| Compensation (push fails twice) | booking kept or cancelled at user's choice; nobody notified | TBD |
| Output schema and status | valid `booking_result`, correct status | TBD |
| Compatibility / regression | passes on a new calendar API version | TBD |

All tests run in `sandbox` mode or with rolled-back transactions, on sandbox calendars, with fictional guests on `example.com` / `example.org`; never against a real calendar, which would email the organizer and guests on every create. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, schema registry and approval UI — TBD (owner: publisher, with Skill Port).
- Whether the target calendar provider honours a "no notifications" mode on all operations — to be confirmed per provider before sandbox tests are trusted.
- Public self-booking by guests (guest-supplied email) needs its own recipe and trust model.
- Recurring meetings and multi-host scheduling are not supported.
- Guest time zones come from the directory and may be stale; the invite always states the zone explicitly.
- Cross-area IDs (contacts, approvals, notifications) must be reconciled with the other recipes in this package.
- Reference-implementation evidence is historical; re-verify on the target runtime before relying on it.
