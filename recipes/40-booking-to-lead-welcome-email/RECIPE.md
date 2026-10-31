# Booking to lead, with a welcome email

## A. Identity and ownership

- **Recipe ID:** `nova.sales.booking-to-lead-welcome-email` (proposed until assigned by Skill Port)
- **Name:** Booking to lead, with a welcome email
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** marketing
- **Tags:** sales, inbound, booking, lead, nurture, welcome-email, pre-approved-scope, business-hours
- **Summary:** When someone books a meeting, the agent creates or updates a sales lead (skipping the owner, staff and the owner's own domains), enrols the lead in an owner-approved inbound nurture campaign, and sends one templated welcome email through an email-sending provider during business hours. A cancellation moves the lead back a stage and stops an unsent welcome. Only this narrow, explicitly pre-approved send is autonomous.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** A booking system that emits `booking.created` and `booking.cancelled` events (reference: the owner's own calendar database); a sales store with leads, campaigns, enrolments, touches and suppression (reference: PostgreSQL behind security-definer dispatcher functions); a transactional email-sending API on a verified sending domain with SPF and DKIM; a scheduled send runner; the owner notification channel; an inbound reply handler that pauses sequences on reply and suppresses on opt-out. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — reference implementation status: Built. In Nova's reference implementation the booking-to-lead bridge, the staff and own-domain guard, the cancellation stage change, auto-enrolment into an approved auto-send inbound campaign and the business-hours welcome send were verified end to end (a rolled-back booking test, then one real send to the owner's own test address). The event-consumer (outbox) form of the bridge and the approval content hash are hardening in this recipe. Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.calendar.bookings` | Capability | Booking events and booking reads | low | UNVERIFIED |
| `nova.calendar.bookings.on_event` | Operation (event, read) | Deliver `booking.created` / `booking.cancelled` from an outbox, at least once | low | UNVERIFIED |
| `nova.sales.policy` | Capability | Owner-defined sales guard rules | low | UNVERIFIED |
| `nova.sales.policy.internal_check` | Operation (read) | Decide whether an attendee is the owner, staff, an own domain or a test booking | low | UNVERIFIED |
| `nova.sales.leads` | Capability | Sales leads | medium | UNVERIFIED |
| `nova.sales.leads.upsert_from_booking` | Operation (create/update) | Create or update a lead by normalised email and link the booking | medium | UNVERIFIED |
| `nova.sales.leads.set_stage` | Operation (update) | Conditional stage change (for example `meeting_booked` → `engaged` on cancel) | low | UNVERIFIED |
| `nova.sales.campaigns` | Capability | Campaigns, steps and their owner approvals | medium | UNVERIFIED |
| `nova.sales.campaigns.resolve_approved` | Operation (read) | Return the single active, approved, auto-send inbound campaign whose content hash matches its approval | low | UNVERIFIED |
| `nova.sales.suppression` | Capability | Opt-outs and do-not-contact list | medium | UNVERIFIED |
| `nova.sales.suppression.check` | Operation (read) | Is this address or domain suppressed? | low | UNVERIFIED |
| `nova.sales.sequences` | Capability | Enrolments and their step schedule | medium | UNVERIFIED |
| `nova.sales.sequences.enroll` | Operation (create) | Enrol a lead once per campaign and booking | medium | UNVERIFIED |
| `nova.sales.sequences.due` | Operation (read) | Enrolments whose next step is due | low | UNVERIFIED |
| `nova.sales.sequences.advance` | Operation (update) | Mark a step sent, skipped or cancelled and schedule the next | low | UNVERIFIED |
| `nova.email.send` | Capability | Transactional email sending on a verified domain | high | UNVERIFIED |
| `nova.email.send.transactional` | Operation (external side effect) | Send one email with an idempotency key | high | UNVERIFIED |
| `nova.sales.touches` | Capability | Record of every sales interaction | low | UNVERIFIED |
| `nova.sales.touches.log` | Operation (create) | Log the sent welcome with the provider message ID | low | UNVERIFIED |
| `nova.notify.owner` | Capability | Deliver a message to the owner's own channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Tell the owner about the new lead and the welcome sent | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.email.send.transactional.provider-api` — one transactional API call on the verified sending domain, plain text plus a minimal HTML part, open and click tracking off, idempotency key supplied. Only this variant is supported; there is no fallback provider or sending domain, because a change of sender identity affects deliverability and consent.
- `nova.calendar.bookings.on_event.outbox` — events are written to an outbox in the booking transaction and consumed afterwards, so a sales failure can never block or roll back a booking. Only this variant is supported (an in-transaction trigger is not an accepted variant).
- All other operations — only the default variant is supported.

**Bindings:** `binding://nova/calendar/booking-events`, `binding://nova/sales/internal-check`, `binding://nova/sales/lead-upsert`, `binding://nova/sales/lead-stage`, `binding://nova/sales/campaign-approved`, `binding://nova/sales/suppression-check`, `binding://nova/sales/seq-enroll`, `binding://nova/sales/seq-due`, `binding://nova/sales/seq-advance`, `binding://nova/email/send`, `binding://nova/sales/touch-log`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: database functions for the bridge and sequence state machine; a scheduled runner that sends through a transactional email API with `secret_ref://email-provider`.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Booking event | `schema://nova/booking-to-lead-welcome-email/booking-event@0.1.0` | owner's calendar | read |
| Lead | `schema://nova/booking-to-lead-welcome-email/lead@0.1.0` | owner's tenant | create, read, update (stage, booking link, next action) |
| Campaign + approval | `schema://nova/booking-to-lead-welcome-email/approved-campaign@0.1.0` | owner's tenant | read |
| Enrolment | `schema://nova/booking-to-lead-welcome-email/enrollment@0.1.0` | one lead + campaign + booking | create, read, update (step state) |
| Touch | `schema://nova/booking-to-lead-welcome-email/touch@0.1.0` | one lead | create |
| Suppression entry | `schema://nova/booking-to-lead-welcome-email/suppression-result@0.1.0` | owner's tenant | read |

**Authorization requirements (declared, never a grant):** `calendar:bookings:read`, `sales:policy:read`, `sales:leads:write`, `sales:campaigns:read`, `sales:suppression:read`, `sales:sequences:write`, `email:send:transactional` (restricted to the approved campaign's welcome step), `sales:touches:write`, `notify:owner`. Provider key as `secret_ref://email-provider`.

**Policy / trust:** risk class `high` — an email reaches a third party without per-message approval. Allowed only inside the pre-approved scope in Section G, which the owner grants once per campaign version.

## C. Intent and applicability

- **User problem:** People who book a meeting should be tracked as leads and get a prompt, professional confirmation, without the owner doing it by hand and without staff or test bookings polluting the pipeline.
- **Desired outcome:** Each genuine external booking produces exactly one up-to-date lead, one enrolment and at most one welcome email, sent in business hours from the approved template; cancellations are reflected.
- **Trigger or entry conditions:** A `booking.created` (status confirmed) or `booking.cancelled` event; a scheduled runner (default every 20 minutes during business hours) for sending.
- **Applicable situations:** Inbound bookings from prospects or clients through the owner's booking page.
- **Out-of-scope situations:** Cold outreach; sending any step not pre-approved; calendar invitations (the booking system sends those); rescheduling logic; replies (handled by an inbound handler and recipe 10); SMS, WhatsApp or calls.
- **Required inputs:** The booking event (Section D) and configuration.
- **Prerequisites:** Exactly one approved inbound nurture campaign with `auto_send = true`; a verified sending domain; guard lists configured; inbound reply and opt-out handling active.
- **Required capabilities:** listed in Section B (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port; every binding is a dependency to resolve before activation.
- **Assumptions:** Verified in the reference implementation: lead creation from bookings, the guard, cancellation stage change, auto-enrolment, the welcome send in daily business hours with `{{first_name}}` and `{{when}}`. Assumed: the booking system can emit events through an outbox.
- **Ambiguity policy:** Use a documented safe default: when anything about the send is uncertain (campaign approval, suppression state, attendee identity), create or update the lead but do not send, and tell the owner.
- **Vedaxi task type:** none — not one of the seven task types supported today.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `booking_event` | `booking-event@0.1.0` | yes | booking system; personal data | — | invalid: reject the event, record `invalid_event` |
| `owner_identities` | emails and domains (owner, staff, own domains) | yes | owner configuration | — | missing: `blocked` (guard cannot run) |
| `business_hours` | `{start:"HH:MM", end:"HH:MM", time_zone, days}` | no | owner | 09:00–20:00 daily, owner time zone | outside 08:00–21:00: reject configuration |
| `daily_cap` | integer 1–200 | no | owner | 50 | clamp |
| `min_lead_time_minutes` | integer 0–1440 | no | owner | 30 | clamp |

```json
{"$id":"schema://nova/booking-to-lead-welcome-email/booking-event@0.1.0","type":"object","required":["event_id","type","booking_id","attendee_email","starts_at","time_zone"],"properties":{"event_id":{"type":"string"},"type":{"enum":["booking.created","booking.cancelled"]},"booking_id":{"type":"string"},"status":{"enum":["confirmed","cancelled"]},"attendee_email":{"type":"string","format":"email","maxLength":254},"attendee_name":{"type":"string","maxLength":120},"event_title":{"type":"string","maxLength":200},"starts_at":{"type":"string","format":"date-time"},"time_zone":{"type":"string"},"source":{"enum":["booking_page","staff_created","api"]},"is_test":{"type":"boolean"},"occurred_at":{"type":"string","format":"date-time"}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `bridge_result` | `run-result@0.1.0` | `event_id`, `status`, `lead_action`, `enrolment`, `welcome` | run record; owner note | personal data; owner only | re-read lead, enrolment and touch |

```json
{"$id":"schema://nova/booking-to-lead-welcome-email/run-result@0.1.0","type":"object","required":["event_id","status","lead_action","enrolment","welcome"],"properties":{"event_id":{"type":"string"},"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},"lead_id":{"type":["string","null"]},"lead_action":{"enum":["created","updated","stage_reverted","skipped_internal","skipped_test","none"]},"enrolment":{"enum":["enrolled","already_enrolled","not_enrolled_no_approved_campaign","not_enrolled_suppressed","pending_step_cancelled","none"]},"welcome":{"enum":["sent","scheduled","skipped_too_late","skipped_suppressed","skipped_cap","not_applicable","failed"]},"provider_message_id":{"type":["string","null"]},"approval_ref":{"type":["string","null"]}}}
```

Other schemas:

```json
{"$id":"schema://nova/booking-to-lead-welcome-email/identity-query@0.1.0","type":"object","required":["email"],"properties":{"email":{"type":"string","maxLength":254},"source":{"type":"string"},"is_test":{"type":"boolean"}}}
{"$id":"schema://nova/booking-to-lead-welcome-email/internal-check-result@0.1.0","type":"object","required":["internal","reason"],"properties":{"internal":{"type":"boolean"},"reason":{"enum":["owner","staff","own_domain","test_booking","staff_created","external"]}}}
{"$id":"schema://nova/booking-to-lead-welcome-email/lead@0.1.0","type":"object","required":["lead_id","email","stage"],"properties":{"lead_id":{"type":"string"},"email":{"type":"string"},"name":{"type":"string"},"first_name":{"type":"string"},"stage":{"enum":["new","contacted","engaged","meeting_booked","attended","proposal","won","lost"]},"source":{"enum":["booking","sourced","manual","other"]},"booking_id":{"type":["string","null"]},"meeting_starts_at":{"type":["string","null"],"format":"date-time"},"next_action":{"type":["string","null"]}}}
{"$id":"schema://nova/booking-to-lead-welcome-email/stage-change@0.1.0","type":"object","required":["lead_id","from_stage","to_stage"],"properties":{"lead_id":{"type":"string"},"from_stage":{"type":"string"},"to_stage":{"type":"string"},"next_action":{"type":["string","null"]},"booking_id":{"type":"string"}}}
{"$id":"schema://nova/booking-to-lead-welcome-email/campaign-query@0.1.0","type":"object","required":["kind"],"properties":{"kind":{"const":"inbound_nurture"}}}
{"$id":"schema://nova/booking-to-lead-welcome-email/approved-campaign@0.1.0","type":"object","required":["campaign_id","status","auto_send","approval"],"properties":{"campaign_id":{"type":"string"},"status":{"const":"active"},"auto_send":{"const":true},"approval":{"type":"object","required":["approval_id","approved_by","approved_at","content_hash","pre_approved_step_ids","hash_matches"],"properties":{"approval_id":{"type":"string"},"approved_by":{"type":"string"},"approved_at":{"type":"string","format":"date-time"},"content_hash":{"type":"string"},"pre_approved_step_ids":{"type":"array","items":{"type":"string"},"minItems":1},"hash_matches":{"const":true}}},"steps":{"type":"array","items":{"type":"object","required":["step_id","subject_template","body_template","wait_hours"],"properties":{"step_id":{"type":"string"},"subject_template":{"type":"string"},"body_template":{"type":"string"},"wait_hours":{"type":"integer","minimum":0}}}}}}
{"$id":"schema://nova/booking-to-lead-welcome-email/suppression-query@0.1.0","type":"object","required":["email"],"properties":{"email":{"type":"string"}}}
{"$id":"schema://nova/booking-to-lead-welcome-email/suppression-result@0.1.0","type":"object","required":["suppressed"],"properties":{"suppressed":{"type":"boolean"},"reason":{"enum":["opt_out","bounce","complaint","manual","none"]}}}
{"$id":"schema://nova/booking-to-lead-welcome-email/enrollment-request@0.1.0","type":"object","required":["lead_id","campaign_id","booking_id"],"properties":{"lead_id":{"type":"string"},"campaign_id":{"type":"string"},"booking_id":{"type":"string"}}}
{"$id":"schema://nova/booking-to-lead-welcome-email/enrollment@0.1.0","type":"object","required":["enrollment_id","lead_id","campaign_id","booking_id","state","next_step_id","next_at"],"properties":{"enrollment_id":{"type":"string"},"lead_id":{"type":"string"},"campaign_id":{"type":"string"},"booking_id":{"type":"string"},"state":{"enum":["active","replied","suppressed","done","cancelled"]},"next_step_id":{"type":["string","null"]},"next_at":{"type":["string","null"],"format":"date-time"},"created":{"type":"boolean"}}}
{"$id":"schema://nova/booking-to-lead-welcome-email/due-query@0.1.0","type":"object","required":["now"],"properties":{"now":{"type":"string","format":"date-time"},"limit":{"type":"integer","minimum":1,"maximum":50}}}
{"$id":"schema://nova/booking-to-lead-welcome-email/rendered-email@0.1.0","type":"object","required":["to","from","reply_to","subject","text","idempotency_key","template_hash"],"properties":{"to":{"type":"string","format":"email"},"from":{"type":"string"},"reply_to":{"type":"string","format":"email"},"subject":{"type":"string","maxLength":150},"text":{"type":"string","maxLength":4000},"html":{"type":"string","maxLength":12000},"idempotency_key":{"type":"string"},"template_hash":{"type":"string"}}}
{"$id":"schema://nova/booking-to-lead-welcome-email/send-result@0.1.0","type":"object","required":["accepted"],"properties":{"accepted":{"type":"boolean"},"provider_message_id":{"type":["string","null"]},"error":{"enum":["invalid_recipient","domain_not_verified","rate_limited","duplicate","provider_error",null]}}}
{"$id":"schema://nova/booking-to-lead-welcome-email/advance-request@0.1.0","type":"object","required":["enrollment_id","step_id","outcome"],"properties":{"enrollment_id":{"type":"string"},"step_id":{"type":"string"},"outcome":{"enum":["sent","skipped","cancelled","failed"]},"reason":{"type":["string","null"]}}}
{"$id":"schema://nova/booking-to-lead-welcome-email/touch@0.1.0","type":"object","required":["lead_id","channel","kind","direction","at"],"properties":{"lead_id":{"type":"string"},"channel":{"const":"email"},"kind":{"enum":["sent"]},"direction":{"const":"out"},"campaign_id":{"type":"string"},"step_id":{"type":"string"},"provider_message_id":{"type":"string"},"at":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/booking-to-lead-welcome-email/touch-receipt@0.1.0","type":"object","required":["touch_id"],"properties":{"touch_id":{"type":"string"},"duplicate":{"type":"boolean"}}}
{"$id":"schema://nova/booking-to-lead-welcome-email/owner-note@0.1.0","type":"object","required":["text"],"properties":{"text":{"type":"string","maxLength":1500}}}
{"$id":"schema://nova/booking-to-lead-welcome-email/delivery@0.1.0","type":"object","required":["delivered"],"properties":{"delivered":{"type":"boolean"},"message_id":{"type":"string"}}}
```

**Example (fictional)**

Event: `{"event_id":"evt_301","type":"booking.created","booking_id":"bk_77","status":"confirmed","attendee_email":"jane@example.com","attendee_name":"Jane Doe","event_title":"Intro call","starts_at":"2026-03-10T20:00:00Z","time_zone":"America/New_York","source":"booking_page","is_test":false}`

Welcome (from the approved template): subject "Looking forward to our call, Jane"; text "Hi Jane, thanks for booking. See you Tue 10 Mar, 4:00 PM (America/New_York). If you'd like, reply with anything you want us to cover. — The team at Example Co. To stop these emails, reply 'stop'."

Result: `{"event_id":"evt_301","status":"completed","lead_id":"ld_912","lead_action":"created","enrolment":"enrolled","welcome":"sent","provider_message_id":"pm_4410","approval_ref":"apr_12"}`

## E. Constraints and budgets

- **Hard constraints:** never create a lead for the owner, staff, own domains, staff-created or test bookings; a booking is never blocked or rolled back by this recipe (outbox consumer only); send only the pre-approved step(s) of the single approved campaign whose current content hash equals the approved hash; send only to the booking's attendee address; at most one welcome per booking; never send to a suppressed address; send only inside business hours and under the daily cap; no LLM personalisation in the auto-send path (the email is exactly the approved template with `{{first_name}}` and `{{when}}` filled); no unresolved placeholders; every email carries an opt-out line; open and click tracking off; stages never move backwards except `meeting_booked` → `engaged` on cancellation.
- **Soft constraints:** send as soon as the window opens; greet by first name; show the meeting time in the attendee's time zone.
- **Resource and tenant scope:** the owner's bookings, leads, campaigns, enrolments and touches.
- **Data sources allowed:** booking events, sales store, guard configuration.
- **External transfers permitted:** recipient address, subject and body to the email-sending provider.
- **Time limit:** event handling ≤ 30 seconds; runner pass ≤ 5 minutes.
- **Cost or token budget:** no LLM; provider cost per email per its plan; daily cap bounds spend.
- **Maximum steps:** 11 (S1–S11).
- **Maximum retries and correction cycles:** store calls 2 retries; send 0 blind retries (reconcile first), at most 1 resend after a confirmed non-delivery; event redelivery is at least once and handled idempotently.
- **Rate limits and concurrency:** runner single-flight; provider rate limits; ≤ 1 email per recipient per 24 hours from this recipe.
- **Retention and logging constraints:** touches and enrolments kept per the operator's sales-data retention (`TBD`); logs carry IDs, not addresses or bodies.
- **Prohibited actions:** sending non-approved steps or edited templates; cold or bulk sends; adding recipients; changing the sending domain; resending after a definite provider acceptance.
- **Stop conditions:** no approved campaign or hash mismatch (lead kept, no send); sending domain unverified; guard configuration missing; owner pauses auto-send.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | lead (external attendee, new email) | guard passed | Allowed | lead readable by email |
| Create | enrolment (once per lead + campaign + booking) | approved campaign; not suppressed | Allowed | unique key holds |
| Create | touch for the sent welcome | provider accepted | Allowed | touch has the provider message ID |
| Read | booking events, guard configuration, campaigns and approvals, suppression, enrolments | own tenant | Allowed | — |
| Update | lead `stage`, `booking_id`, `meeting_starts_at`, `next_action` | forward moves only, plus `meeting_booked` → `engaged` on cancel | Allowed | re-read |
| Update | enrolment step state | sent, skipped, cancelled, failed | Allowed | re-read |
| Update | campaign, templates, approvals, suppression | — | Not Allowed | — |
| Delete | any | — | Not Allowed | — |
| External side effect | welcome email to the attendee | pre-approved scope (Section G) | Allowed within the pre-approved scope; otherwise Approval Required | provider acceptance + touch |
| External side effect | owner note | after a send or a skip that needs attention | Allowed | delivery receipt |

- **Idempotency:** events keyed by `event_id`; lead keyed by normalised email; enrolment keyed by (`lead_id`, `campaign_id`, `booking_id`); send keyed by hash(`booking_id`, `step_id`) passed to the provider and stored before the call.
- **Expected side effects:** a lead and enrolment rows; one email delivered to the attendee; provider charges.
- **Compensation:** an unsent welcome can be cancelled; a sent email cannot be recalled — the owner can follow up personally; a wrongly created lead can be marked `lost` with a reason by the owner.
- **Irreversible:** the sent email.
- **Ambiguous outcomes:** if the send call times out, query the provider (or the stored key) for a message with the idempotency key before any resend; never send twice on an unknown outcome.

## G. Autonomy and approval policy

- **Default autonomy level:** A2 — bounded autonomy inside an explicit pre-approved scope.
- **Planning level:** A2.
- **Execution level:** A2 for leads and enrolment; A2 for the welcome send only inside the pre-approved scope below.
- **Verification level:** A2.
- **Recovery level:** A1 — no resend without reconciliation; anything outside scope goes to the owner.
- **Pre-approved scope (granted by the owner once per campaign version, recorded as an approval with a content hash):** campaign kind `inbound_nurture`, `auto_send = true`, status `active`; only the step IDs listed in the approval (default: the welcome step); recipient = the attendee of a confirmed external booking; once per booking; inside business hours; under the daily cap; template unchanged since approval. Editing any template or step invalidates the approval automatically.
- **Approval required for:** any other step or campaign; any send outside business hours; raising the cap; a new campaign version.
- **Actions never permitted:** cold outreach from this recipe; sending to internal or test attendees; sending to suppressed addresses; LLM-written content in the auto-send path.
- **Conditions requiring clarification:** two active approved inbound campaigns (which one?).
- **Conditions requiring escalation:** approval hash mismatch; sending domain unverified; repeated provider rejections; bounce or complaint on a welcome.
- **Safe state after stop, cancellation or escalation:** leads and enrolments may exist; no email is sent until the condition is resolved; pending steps stay pending or are cancelled, never sent.

## H. Execution plan

**S1 — Receive and validate the booking event**
1. Objective: a valid, not-yet-processed event.
2. Capability: `nova.calendar.bookings.on_event.outbox`.
3. Input mapping: outbox record → `booking-event@0.1.0`.
4. Dependencies: none.
5. Preconditions: guard configuration present.
6. Action / side effects: validate; record `event_id` as seen; no external effect.
7. Autonomy: A2; no approval.
8. Timeout / retry: 5 s / redelivery by the outbox.
9. Idempotency: `event_id` processed once; duplicates acknowledged without work.
10. Postconditions: a validated event.
11. Verification: schema valid; `status` consistent with `type`.
12. Failure routes: invalid → `failed` for this event, `invalid_event` recorded, event acknowledged (no infinite redelivery).
13. Escalation triggers: more than 5 invalid events per day → owner note.
14. Evidence: event ID and validation result.

**S2 — Internal and test guard**
1. Objective: keep the owner, staff, own domains and tests out of the pipeline.
2. Capability: `nova.sales.policy.internal_check`.
3. Input mapping: `attendee_email`, `source`, `is_test`.
4. Dependencies: S1.
5. Preconditions: event valid.
6. Action / side effects: read-only check: exact owner or staff address, own domain or subdomain, `is_test`, `source = staff_created`.
7. Autonomy: A2.
8. Timeout / retry: 5 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: `internal-check-result@0.1.0`.
11. Verification: domain match on the full domain after `@`, case-insensitive, subdomains included.
12. Failure routes: internal → stop, `completed` with `skipped_internal` / `skipped_test`; check unavailable → `blocked` (never assume external).
13. Escalation triggers: none.
14. Evidence: reason code.

**S3 — Upsert the lead (created events)**
1. Objective: one current lead for this attendee.
2. Capability: `nova.sales.leads.upsert_from_booking`.
3. Input mapping: normalised email (trimmed, lower-case), name, first name (first token of the name; empty if unknown), `booking_id`, `starts_at`.
4. Dependencies: S2 (external).
5. Preconditions: `type = booking.created`.
6. Action / side effects: create with `source = booking`, stage `meeting_booked`; or update: `new`, `contacted`, `engaged`, `lost` → `meeting_booked` (lost is reopened with a note); `meeting_booked` stays; `attended`, `proposal`, `won` keep their stage and only link the booking.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: unique on normalised email; repeated events do not create duplicates.
10. Postconditions: `lead@0.1.0` with the booking linked.
11. Verification: exactly one lead for the email; stage never moved backwards.
12. Failure routes: store error → event redelivered (S1 idempotency prevents double effects).
13. Escalation triggers: none.
14. Evidence: lead ID and action.

**S4 — Cancellation handling (cancelled events)**
1. Objective: reflect the cancellation and stop an unsent welcome.
2. Capability: `nova.sales.leads.set_stage`; `nova.sales.sequences.advance` (outcome `cancelled`).
3. Input mapping: `booking_id`, lead by email.
4. Dependencies: S2.
5. Preconditions: `type = booking.cancelled`; lead exists.
6. Action / side effects: if stage is `meeting_booked` and linked to this booking → `engaged` with `next_action = rebook`; cancel any not-yet-sent step of the enrolment for this booking. No email is sent.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: conditional on the current stage and step state.
10. Postconditions: stage and step state updated, or unchanged with a reason (later stage, other booking).
11. Verification: re-read lead and enrolment.
12. Failure routes: lead not found → `completed`, nothing to do; store error → redelivery.
13. Escalation triggers: a welcome already sent for this booking → owner note (the owner may want to follow up personally).
14. Evidence: stage change and cancelled step IDs.

**S5 — Resolve the approved campaign**
1. Objective: find the one campaign this lead may be auto-enrolled into.
2. Capability: `nova.sales.campaigns.resolve_approved`.
3. Input mapping: `kind = inbound_nurture`.
4. Dependencies: S3.
5. Preconditions: lead created or updated.
6. Action / side effects: read-only.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: `approved-campaign@0.1.0` with `hash_matches = true`, or none.
11. Verification: exactly one active campaign; approval present; current content hash equals the approved hash; welcome step is in `pre_approved_step_ids`.
12. Failure routes: none or hash mismatch → lead kept, `not_enrolled_no_approved_campaign`, `partially_completed`; two campaigns → clarification (Section G).
13. Escalation triggers: hash mismatch → owner note asking to re-approve.
14. Evidence: campaign ID, approval ID, hash comparison.

**S6 — Suppression and duplicate check**
1. Objective: never enrol or email someone who opted out or is already enrolled for this booking.
2. Capability: `nova.sales.suppression.check`.
3. Input mapping: attendee email and its domain.
4. Dependencies: S5.
5. Preconditions: approved campaign found.
6. Action / side effects: read-only.
7. Autonomy: A2.
8. Timeout / retry: 5 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: `suppressed = false`, or a reason.
11. Verification: check includes opt-outs, bounces and complaints.
12. Failure routes: suppressed → `not_enrolled_suppressed`, `completed` without send; check unavailable → `blocked` for the send part (never assume not suppressed).
13. Escalation triggers: none.
14. Evidence: suppression result (reason only).

**S7 — Enrol**
1. Objective: one enrolment with the welcome due now.
2. Capability: `nova.sales.sequences.enroll`.
3. Input mapping: `lead_id`, `campaign_id`, `booking_id`.
4. Dependencies: S6.
5. Preconditions: not suppressed.
6. Action / side effects: create the enrolment with `next_step_id` = welcome, `next_at = now`; a suppressed or replied lead is never reactivated.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: unique on (`lead_id`, `campaign_id`, `booking_id`); a repeat returns `created = false`.
10. Postconditions: `enrollment@0.1.0`, state `active`.
11. Verification: one enrolment for the key.
12. Failure routes: store error → redelivery.
13. Escalation triggers: none.
14. Evidence: enrolment ID.

**S8 — Send-window gate and due selection (runner)**
1. Objective: send only inside business hours and under the cap.
2. Capability: `nova.sales.sequences.due`.
3. Input mapping: `now`, business hours, daily cap.
4. Dependencies: S7 (the runner picks up due enrolments).
5. Preconditions: now inside business hours; sends today below the cap; auto-send not paused.
6. Action / side effects: read due steps for this recipe's approved campaign only.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: due items, each with lead, step and meeting time.
11. Verification: every item's step is pre-approved; enrolment state `active`.
12. Failure routes: outside hours → wait for the next window (`scheduled`); cap reached → `skipped_cap` until tomorrow, owner note; meeting starts within `min_lead_time_minutes` or has passed → advance `skipped`, reason `too_late`.
13. Escalation triggers: cap reached on two consecutive days.
14. Evidence: due IDs and gate decision.

**S9 — Render the welcome email**
1. Objective: exactly the approved template, filled.
2. Capability: local deterministic renderer.
3. Input mapping: step templates; `{{first_name}}` (fallback "there"); `{{when}}` = meeting start in the attendee's time zone, for example "Tue 10 Mar, 4:00 PM (America/New_York)".
4. Dependencies: S8.
5. Preconditions: approval hash still matches.
6. Action / side effects: render subject, text and minimal HTML; append the opt-out line and sender details; compute the template hash and the idempotency key.
7. Autonomy: A2.
8. Timeout / retry: 2 s / none.
9. Idempotency: deterministic.
10. Postconditions: `rendered-email@0.1.0`.
11. Verification: no `{{` or `}}` left; template hash equals the approved hash; recipient equals the attendee email; opt-out line present.
12. Failure routes: any check fails → advance `failed` with the reason, no send, owner note.
13. Escalation triggers: hash mismatch.
14. Evidence: template hash and key.

**S10 — Send the welcome**
1. Objective: one delivery attempt that cannot double-send.
2. Capability: `nova.email.send.transactional.provider-api`.
3. Input mapping: rendered email.
4. Dependencies: S9.
5. Preconditions: suppression re-checked within the last 5 minutes; key not already marked sent.
6. Action / side effects: one email to the attendee; provider charge.
7. Autonomy: A2 within the pre-approved scope.
8. Timeout / retry: 20 s / no blind retry.
9. Idempotency: provider idempotency key = hash(`booking_id`, `step_id`); key stored before the call.
10. Postconditions: `send-result@0.1.0` with a provider message ID.
11. Verification: `accepted = true` and a message ID.
12. Failure routes: timeout → look up by key; found → treat as sent; not found → one resend with the same key; invalid recipient → advance `failed`, add a bounce-type suppression only via the inbound handler, owner note; domain not verified → stop the runner, `blocked`.
13. Escalation triggers: two provider rejections in a run; any complaint signal.
14. Evidence: provider message ID.

**S11 — Log, advance and tell the owner**
1. Objective: records match reality and the owner knows.
2. Capability: `nova.sales.touches.log`; `nova.sales.sequences.advance`; `nova.notify.owner.send`.
3. Input mapping: lead, step, provider message ID.
4. Dependencies: S10 (or a skip or failure route).
5. Preconditions: an outcome exists.
6. Action / side effects: log the touch (`email`, `sent`, `out`); advance the step (`sent` → next step or `done`); send a short owner note ("New lead from booking: Jane Doe, Tue 10 Mar 4:00 PM — welcome sent").
7. Autonomy: A2.
8. Timeout / retry: 10 s / 3 retries each.
9. Idempotency: touch unique on provider message ID; advance conditional on the step's current state; note key = `event_id`.
10. Postconditions: touch logged, step advanced, owner told.
11. Verification: re-read the enrolment; touch exists.
12. Failure routes: touch or advance write fails → retry; still failing → the stored send key prevents a resend on the next runner pass; run `partially_completed`.
13. Escalation triggers: none.
14. Evidence: touch ID, enrolment state, note ID.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Event validity | event matches `booking-event@0.1.0` | validator | valid | reject event |
| Guard | no lead for owner, staff, own domains or tests | S2 + lead store | none created | delete nothing; mark the lead `lost` with reason `internal` via owner, report defect |
| Booking safety | booking committed independently of this recipe | outbox design | always | not applicable at runtime; design check |
| Stage monotonicity | no backwards move except cancel rule | before/after stage | holds | revert and report |
| Approval binding | template hash = approved hash; step pre-approved | S5, S9 | equal | no send |
| Suppression | not suppressed at S6 and again at S10 | suppression store | not suppressed | no send |
| Window and cap | send time in hours; count ≤ cap | runner log | holds | no send |
| Single send | one provider message per idempotency key | key store, provider lookup | one | report the duplicate |
| Content | rendered text has no placeholders and has the opt-out line | S9 | holds | no send |
| Records | touch and step state match the provider result | S11 | match | reconcile |

Provider acceptance proves hand-off, not inbox delivery; bounces and complaints arriving later are processed by the inbound handler and suppress the address.

## J. Self-correction and recovery

- **Failure classes:** invalid event, guard unavailable, store error, no approved campaign, hash mismatch, suppression check unavailable, outside window, cap reached, render check failure, provider timeout, provider rejection, record failure.
- **Permitted corrections:** idempotent reprocessing of redelivered events; waiting for the next window; one resend with the same key after a confirmed non-delivery; retrying record writes.
- **Forbidden corrections:** sending without a matching approval; using an LLM to "fix" a template; switching provider or sending domain; skipping the suppression check; reactivating suppressed or replied enrolments.
- **Maximum correction cycles:** 1 per event and 1 per send.
- **Reverification required after correction:** yes — S9 checks before any resend; S5 hash check before any send after a campaign edit.
- **Stop conditions:** domain unverified; approval invalid; owner pause.
- **Recovery or compensation procedure:** cancel unsent steps; the owner follows up personally on a sent email that should not have gone.
- **What must be reported to the user:** new leads, sends, skips with reasons, approval problems, provider failures.

## K. Independent challenge

- **Assumptions to challenge:** that the attendee is a prospect (could be a supplier, candidate or friend); that the booking address belongs to the attendee.
- **Alternative approach:** for owners who prefer a personal touch, enrol but hold the welcome as a draft for one-click approval (A1 mode) instead of auto-send.
- **Contradictory evidence:** repeated cancellations from the same address, replies saying "wrong person", bounces.
- **Could mislead:** "welcome sent" when the provider accepted but later bounced; the owner note is amended when a bounce arrives.
- **Separate verifier:** a daily reconciliation compares bookings, leads, enrolments and touches for the day and flags gaps (booking without lead, lead without enrolment, two sends for one booking).
- **Findings requiring disclosure:** guard misses (an internal lead found), approval mismatches, duplicate sends.

## L. Human escalation

- **Triggers:** approval missing or invalid; two approved campaigns; sending domain unverified; repeated provider rejections; complaint or bounce; cap reached two days running; a welcome already sent for a cancelled booking.
- **Why the agent cannot decide:** content approval, sender identity and contact with real people are the owner's decisions.
- **Information presented:** lead, booking time, campaign and approval state, provider error, counts.
- **Decision requested:** re-approve the campaign version; choose the campaign; fix the domain; raise or keep the cap; follow up personally.
- **Options and consequences:** re-approving resumes auto-send for the listed steps; declining keeps leads without emails.
- **Safe behaviour while waiting:** leads and enrolments continue; no sends.
- **If no response:** pending welcomes become `skipped/too_late` after the meeting starts; nothing is sent late.
- **Resume:** S5 and S8 re-check approval, window and cap before any send.

## M. Completion conditions

Mandatory: event processed once; guard respected; lead and enrolment consistent; at most one welcome per booking, only within the pre-approved scope; touch logged for every send; owner informed of exceptions; evidence references present.

- `completed` — the event was handled fully: internal skip, cancellation applied, or lead + enrolment + welcome sent (or correctly skipped as suppressed or too late).
- `partially_completed` — lead created but no approved campaign, cap reached, owner note undelivered, or records still reconciling after a send.
- `blocked` — guard or suppression check unavailable, guard configuration missing, or sending domain unverified.
- `failed` — invalid event, or the send was rejected and could not be corrected.
- `cancelled` — the owner paused auto-send or the booking was cancelled before the welcome went out (welcome step cancelled).

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome; `run-result@0.1.0`; step statuses S1–S11; verification results and failed checks; evidence references (event ID, lead ID, enrolment ID, approval ID, provider message ID, touch ID); approvals and scope (pre-approved campaign version); side effects (email sent, charges) and reconciliation status; corrections and retries; unresolved challenges; limitations; escalation state; learning candidate. Never include provider keys; addresses appear only in the owner's private view.

## O. Learning and revision

- **Outcome metrics:** bookings → leads conversion, guard skips by reason, welcome send rate, time from booking to welcome, bounces and complaints per 100 sends, reply rate to the welcome, cancellation rate, meeting attendance after a welcome.
- **Evidence retained:** run records, touches and enrolments per retention policy.
- **Data minimisation:** metrics carry counts; addresses are not copied into metrics.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}`.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review; any widening of the pre-approved scope requires a new owner approval.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (sandbox calendar, tester-owned inbox) | lead `meeting_booked`, enrolled, one welcome in hours | UNVERIFIED on Skill Port; observed in the reference implementation (rolled-back booking test plus one send to the owner's own address) |
| Invalid input (event missing `attendee_email`) | rejected, acknowledged | TBD |
| Missing input (guard configuration absent) | `blocked` | TBD |
| Missing capability (email binding absent) | lead and enrolment created; send `blocked` | TBD |
| Permission denied (`email:send:transactional` missing) | no send; owner note | TBD |
| Unauthorized scope (send a non-pre-approved step) | refused | TBD |
| Tenant isolation | another tenant's booking never creates a lead here | TBD |
| Guard (owner, staff, own domain, subdomain, test, staff-created) | no lead | TBD |
| Cancellation before send | stage `engaged`, welcome cancelled | TBD |
| Cancellation after send | stage `engaged`, owner note, no further email | TBD |
| Later-stage lead books again | stage unchanged, booking linked | TBD |
| Approval hash mismatch (template edited) | no send; owner asked to re-approve | TBD |
| Suppressed attendee | no enrolment, no send | TBD |
| Outside business hours | sent at window open | TBD |
| Too late (meeting in 10 minutes) | skipped `too_late` | TBD |
| Timeout on send, provider has the key | treated as sent; no duplicate | TBD |
| Duplicate / replay (event delivered twice) | one lead, one enrolment, one email | TBD |
| False success (provider accepts, later bounce) | owner note amended; address suppressed by inbound handler | TBD |
| Failed verification (unresolved placeholder) | no send | TBD |
| Correction budget exhausted | step `failed`, owner note | TBD |
| Escalation (two approved campaigns) | clarification; no send | TBD |
| Cancellation (owner pauses auto-send) | no sends; leads continue | TBD |
| Compensation (cancel unsent step) | step `cancelled`; nothing sent | TBD |
| Output schema and status | valid `run-result` | TBD |
| Compatibility / regression (provider API version change) | idempotency key honoured | TBD |

Never test with real bookings on a live calendar: a persisted test booking can trigger real invitations and emails to real people. Use rolled-back transactions or a sandbox calendar and a tester-owned inbox. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, the approval object format and how approvals are bound to content hashes — TBD (owner: publisher, with Skill Port).
- Business-hours default and daily cap per owner — TBD (owner: operator).
- Lawful basis for the welcome email (transactional confirmation vs marketing) depends on jurisdiction and content; the template must stay transactional in tone, and the owner confirms the basis at approval.
- Whether later nurture steps should ever be pre-approved — currently no (welcome only by default).
- Physical sender address and opt-out wording must meet local email law — owner supplies.
- Bounce and complaint handling depends on the inbound handler, which is outside this recipe.
