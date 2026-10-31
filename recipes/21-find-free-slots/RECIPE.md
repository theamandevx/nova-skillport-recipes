# Find free slots

## A. Identity and ownership

- **Recipe ID:** `nova.calendar.find-free-slots` (proposed until assigned by Skill Port)
- **Name:** Find free slots
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** productivity
- **Tags:** calendar, availability, scheduling, slot-engine, busy-store, time-zones, MCP, read-only · Vedaxi task type: none of the seven supported today
- **Summary:** Given a meeting length and a date range, the agent computes bookable slots from one master busy-store (bookings, manual blocks, busy time synced from connected calendars and busy time parsed from meeting invites), honouring working hours, notice, horizon, buffers and daily limits, and returns them in the requester's time zone with UTC and a source-freshness disclosure. The same operation is exposed to external booking surfaces as an MCP tool.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** A calendar database that is the master busy-store, with a slot-engine function; an MCP server exposing `get_slots` and `get_availability`; at least one busy-sync source (a connected calendar such as Google Calendar, and/or a mailbox from which invitation subjects are read); an IANA time-zone database. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — the slot engine, working hours that cross midnight, buffers, daily limits, calendar busy-sync with travel padding and invite parsing have run in Nova's reference implementation. Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.calendar.availability` | Capability | Answer "when am I free?" from the master busy-store | low | UNVERIFIED |
| `nova.calendar.availability.get_slots` | Operation (read) | Return bookable start/end pairs for a length and range (MCP tool `get_slots`) | low | UNVERIFIED |
| `nova.calendar.availability.get_working_hours` | Operation (read) | Return working-hour windows, owner time zone and scheduling policy (MCP tool `get_availability`) | low | UNVERIFIED |
| `nova.calendar.availability.list_calendar` | Operation (read) | Return every booking and busy block in a range (used to verify slots independently) | medium | UNVERIFIED |
| `nova.calendar.busy_sync` | Capability | Keep the busy-store aligned with connected calendars and received invites | medium | UNVERIFIED |
| `nova.calendar.busy_sync.status` | Operation (read) | Last successful sync time, last error and item count per source | low | UNVERIFIED |
| `nova.calendar.busy_sync.sync_calendar` | Operation (create / update / delete busy blocks of one source) | Mirror a connected calendar's busy events for the full future window | medium | UNVERIFIED |
| `nova.calendar.busy_sync.sync_invites` | Operation (read mailbox metadata; create / update busy blocks) | Parse meeting times from invitation subjects and block them | medium | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.calendar.availability.get_slots.default` — the owner's own agent. May pass a `min_notice` override (including 0) because the owner may book their own time at short notice. Selected only when the runtime authenticates the caller as the calendar owner.
- `nova.calendar.availability.get_slots.public` — any external caller (public booking page, another assistant through MCP). The server strips `min_notice`, `host` and horizon overrides before the slot engine runs, and returns free/busy only (no titles, attendees or sources). Selected for every caller that is not the owner. **There is no fallback between the two variants**: a public call never falls back to `default`, because that would widen what an outsider can see and book.
- `nova.calendar.busy_sync.sync_calendar.default` — calendar API read of the next 30 days (single events expanded, up to 250 items), classified then sent as one full-window payload. Only the default variant is supported.
- `nova.calendar.busy_sync.sync_invites.default` — reads invitation subject lines and dates only (no bodies), parses the meeting time and named time-zone abbreviations through a documented mapping table, groups by meeting title (latest message wins, which handles reschedules and cancellations) and skips all-day items. Only the default variant is supported.
- `get_working_hours`, `list_calendar` and `status` — only the default variant is supported (read-only).

**Bindings:** `binding://nova/calendar/get-slots`, `binding://nova/calendar/get-availability`, `binding://nova/calendar/calendar-range`, `binding://nova/calendar/busy-sync-status`, `binding://nova/calendar/busy-sync-calendar`, `binding://nova/calendar/busy-sync-invites` — all `status: UNVERIFIED`. Reference implementation: a PostgreSQL schema with security-definer functions (`slots`, `availability_get`, `calendar`, a sync function per source) called through a stateless MCP server on an n8n automation hub (Streamable HTTP, JSON-RPC, bearer-token authentication) and through the owner's assistant; busy-sync runs as scheduled workflows every 30 minutes.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Working-hour window | `schema://nova/find-free-slots/working-hours@0.1.0` | the calendar owner | read |
| Scheduling policy | `schema://nova/find-free-slots/policy@0.1.0` | the calendar owner | read |
| Booking (times only) | `schema://nova/find-free-slots/busy-interval@0.1.0` | the owner's tenant | read |
| Busy block | `schema://nova/find-free-slots/busy-interval@0.1.0` | the owner's tenant, one `source` each | read; create / update / delete only through a full-window sync of the same source |
| Sync status | `schema://nova/find-free-slots/sync-status@0.1.0` | the owner's tenant | read |
| Slot | `schema://nova/find-free-slots/slot-result@0.1.0` | computed, not stored | — |

**Authorization requirements (declared, never a grant):** `calendar:availability:read` (both variants); `calendar:busy:read_detail` (owner variant only — titles and sources); `calendar:busy_sync:write` (owner variant only, own sources); `mail:metadata:read` (invite parsing, subjects and dates only); `calendar:external:read` (connected calendar). Connected-calendar and mailbox credentials live in the runtime as `secret_ref://calendar-connection` and `secret_ref://mailbox-connection`; the MCP bearer token as `secret_ref://calendar-mcp-token`.

**Policy / trust:** risk class `low` for reads, `medium` for busy-sync writes (a bad sync can hide or invent busy time). Public callers are untrusted: their arguments cannot change policy, and the MCP layer, not the client, enforces the stripping of overrides.

## C. Intent and applicability

- **User problem:** The user (or someone booking with the user) needs to know when the user is genuinely free for a meeting of a given length, across time zones, without double-booking against meetings that live in other calendars or in unaccepted invites.
- **Desired outcome:** A short, correct list of bookable slots in the requester's time zone, each also given in UTC and in the owner's time zone, with an honest note if any busy source is stale.
- **Trigger or entry conditions:** The user asks "when am I free…", "find 30 minutes with John next week", or an external booking surface calls the MCP tool `get_slots`.
- **Applicable situations:** Proposing times in a chat or email draft; feeding the "Book a meeting" recipe; powering a public booking page.
- **Out-of-scope situations:** Creating, moving or cancelling bookings (recipes 22 and 23); changing working hours or policy; multi-host or round-robin scheduling (single host by design); reading other people's calendars; writing to any connected external calendar.
- **Required inputs:** `duration_min` or `event_type`, `from`, `to` (see Section D).
- **Prerequisites:** Working hours configured; owner time zone set; the busy-store reachable; at least one busy source synced within its freshness threshold, or the caller accepts a disclosed degraded result.
- **Required capabilities:** `nova.calendar.availability`, `nova.calendar.busy_sync` (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port; every binding is a dependency to resolve before activation.
- **Assumptions:** Verified in the reference implementation: wall-time to UTC conversion through the time-zone database, working-hour windows that cross midnight (stored as an end minute above 1,440), per-event-type buffers around bookings, a daily booking cap, half-open intervals (a slot may start exactly when a block ends), travel padding for in-person events, invite-subject parsing. Assumed: the target runtime provides an IANA time-zone database and an equivalent calendar API.
- **Ambiguity policy:** Use a documented safe default where one exists (display in the requester's own time zone if known, else the owner's, and say which); ask when the time zone or range is genuinely ambiguous (for example "Tuesday" when two Tuesdays are in range); never guess a working-hours change.
- **Vedaxi task type:** none of the seven supported today.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `duration_min` | integer 15–240, multiple of 5 | one of these two | user or caller | — | Both missing: ask once. Both present and inconsistent: `event_type` wins and the run says so. |
| `event_type` | string slug from the owner's event types (for example `intro` = 30 min, `consult` = 60 min) | one of these two | user or caller | — | Unknown slug: reject and list the valid slugs. |
| `from` | date `YYYY-MM-DD` or date-time with offset | yes | user or caller | today | In the past: clamp to now and say so. |
| `to` | date or date-time with offset, ≥ `from` | yes | user or caller | `from` + 7 days | Beyond the horizon: clamp to the horizon and say so. Before `from`: reject. |
| `display_tz` | IANA zone name (for example `Europe/London`) | no | user or caller | requester's known zone, else owner's zone | Abbreviation such as "EST" or "IST": ambiguous — map only through the documented table, else ask. Unknown zone: reject. |
| `max_results` | integer 1–50 | no | caller | 10 | Out of range: clamp. |
| `local_window` | `{earliest:"HH:MM", latest:"HH:MM"}` in `display_tz` | no | user | none | `earliest ≥ latest`: reject. |
| `min_notice_min` | integer ≥ 0 | no | owner variant only | policy value | Present on a public call: silently stripped by the server; never honoured. |

```json
{"$id":"schema://nova/find-free-slots/request@0.1.0","type":"object","required":["from","to"],
 "oneOf":[{"required":["duration_min"]},{"required":["event_type"]}],
 "properties":{"duration_min":{"type":"integer","minimum":15,"maximum":240,"multipleOf":5},
 "event_type":{"type":"string","pattern":"^[a-z0-9-]{2,40}$"},"from":{"type":"string"},"to":{"type":"string"},
 "display_tz":{"type":"string","pattern":"^[A-Za-z_]+(/[A-Za-z0-9_+-]+)+$|^UTC$"},
 "max_results":{"type":"integer","minimum":1,"maximum":50},
 "local_window":{"type":"object","properties":{"earliest":{"type":"string","pattern":"^[0-2][0-9]:[0-5][0-9]$"},"latest":{"type":"string","pattern":"^[0-2][0-9]:[0-5][0-9]$"}}},
 "min_notice_min":{"type":"integer","minimum":0}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `slot_result` | object `schema://nova/find-free-slots/slot-result@0.1.0` | `status`, `display_tz`, `owner_tz`, `slots`, `policy_applied`, `freshness` | the requesting conversation or the MCP response | owner variant: may include source names; public variant: free/busy only | every slot re-checked against `list_calendar` (Section I) |

```json
{"$id":"schema://nova/find-free-slots/slot-result@0.1.0","type":"object",
 "required":["status","display_tz","owner_tz","slots","policy_applied","freshness"],
 "properties":{"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},
 "display_tz":{"type":"string"},"owner_tz":{"type":"string"},
 "slots":{"type":"array","maxItems":50,"items":{"type":"object","required":["start_utc","end_utc","start_display","start_owner"],
  "properties":{"start_utc":{"type":"string","format":"date-time"},"end_utc":{"type":"string","format":"date-time"},
  "start_display":{"type":"string"},"end_display":{"type":"string"},"start_owner":{"type":"string"},
  "flags":{"type":"array","items":{"enum":["dst_transition_day","crosses_midnight_display","crosses_midnight_owner"]}}}}},
 "policy_applied":{"type":"object","properties":{"min_notice_min":{"type":"integer"},"horizon_days":{"type":"integer"},
  "granularity_min":{"type":"integer"},"buffer_min":{"type":"integer"},"max_bookings_per_day":{"type":["integer","null"]}}},
 "freshness":{"type":"object","required":["overall"],"properties":{"overall":{"enum":["fresh","degraded","stale"]},
  "sources":{"type":"array","items":{"type":"object","properties":{"source":{"type":"string"},
  "last_success_at":{"type":["string","null"],"format":"date-time"},"state":{"enum":["fresh","degraded","stale","never"]}}}}}},
 "notes":{"type":"array","items":{"type":"string"}}}}
```

Other schemas:

```json
{"$id":"schema://nova/find-free-slots/working-hours@0.1.0","type":"object","required":["owner_tz","windows"],
 "properties":{"owner_tz":{"type":"string"},"windows":{"type":"array","items":{"type":"object","required":["weekday","start_min","end_min"],
 "properties":{"weekday":{"type":"integer","minimum":0,"maximum":6},"start_min":{"type":"integer","minimum":0,"maximum":1439},
 "end_min":{"type":"integer","minimum":1,"maximum":2880}}}}}}
{"$id":"schema://nova/find-free-slots/policy@0.1.0","type":"object","required":["min_notice_min","horizon_days","granularity_min"],
 "properties":{"min_notice_min":{"type":"integer","minimum":0},"horizon_days":{"type":"integer","minimum":1,"maximum":365},
 "granularity_min":{"enum":[5,10,15,30,60]},"buffer_min":{"type":"integer","minimum":0},"max_bookings_per_day":{"type":["integer","null"],"minimum":1}}}
{"$id":"schema://nova/find-free-slots/busy-interval@0.1.0","type":"object","required":["start_utc","end_utc","source"],
 "properties":{"start_utc":{"type":"string","format":"date-time"},"end_utc":{"type":"string","format":"date-time"},
 "source":{"enum":["booking","manual","calendar","invite"]},"ext_ref":{"type":["string","null"]},"label":{"type":["string","null"]},
 "in_person":{"type":"boolean"}}}
{"$id":"schema://nova/find-free-slots/sync-status@0.1.0","type":"object","required":["source","last_success_at"],
 "properties":{"source":{"type":"string"},"last_success_at":{"type":["string","null"],"format":"date-time"},
 "last_attempt_at":{"type":["string","null"],"format":"date-time"},"last_error":{"type":["string","null"]},"items":{"type":"integer","minimum":0}}}
{"$id":"schema://nova/find-free-slots/sync-payload@0.1.0","type":"object","required":["source","window_from","window_to","events"],
 "properties":{"source":{"type":"string"},"window_from":{"type":"string","format":"date-time"},"window_to":{"type":"string","format":"date-time"},
 "events":{"type":"array","minItems":1,"items":{"$ref":"schema://nova/find-free-slots/busy-interval@0.1.0"}}}}
{"$id":"schema://nova/find-free-slots/owner-ref@0.1.0","type":"object","required":["owner_id"],"properties":{"owner_id":{"type":"string"}}}
{"$id":"schema://nova/find-free-slots/range@0.1.0","type":"object","required":["from_utc","to_utc"],
 "properties":{"from_utc":{"type":"string","format":"date-time"},"to_utc":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/find-free-slots/invite-metadata@0.1.0","type":"object","required":["message_id","subject","received_at"],
 "properties":{"message_id":{"type":"string"},"subject":{"type":"string","maxLength":500},"received_at":{"type":"string","format":"date-time"},
 "sender_domain":{"type":"string"}}}
```

Interval rule: `end_utc > start_utc` always; intervals are half-open `[start, end)`. A zero-length block is invalid (it would block nothing and hide a data error).

**Example (fictional)** — owner Jane Doe works Monday–Friday 09:00–17:00 `America/New_York`; John Roe is in London.

Input: `{"duration_min":30,"from":"2026-11-03","to":"2026-11-04","display_tz":"Europe/London","max_results":3}`

Output: `{"status":"completed","display_tz":"Europe/London","owner_tz":"America/New_York","slots":[{"start_utc":"2026-11-03T14:00:00Z","end_utc":"2026-11-03T14:30:00Z","start_display":"Tue 3 Nov 14:00 GMT","end_display":"14:30 GMT","start_owner":"Tue 3 Nov 09:00 EST"},{"start_utc":"2026-11-03T15:30:00Z","end_utc":"2026-11-03T16:00:00Z","start_display":"Tue 3 Nov 15:30 GMT","end_display":"16:00 GMT","start_owner":"Tue 3 Nov 10:30 EST"},{"start_utc":"2026-11-04T14:00:00Z","end_utc":"2026-11-04T14:30:00Z","start_display":"Wed 4 Nov 14:00 GMT","end_display":"14:30 GMT","start_owner":"Wed 4 Nov 09:00 EST"}],"policy_applied":{"min_notice_min":480,"horizon_days":30,"granularity_min":30,"buffer_min":10,"max_bookings_per_day":null},"freshness":{"overall":"fresh","sources":[{"source":"calendar:work","last_success_at":"2026-11-02T13:30:00Z","state":"fresh"},{"source":"invite:mailbox","last_success_at":"2026-11-02T13:35:00Z","state":"fresh"}]},"notes":["New York is 5 hours behind London from 1 Nov 2026 (it was 4 hours from 25 Oct to 31 Oct)."]}`

## E. Constraints and budgets

- **Hard constraints:** never return a slot that overlaps a booking (plus its buffer), a manual block, a synced calendar block or an invite block; never return a slot that starts before now + minimum notice or after the horizon; never return a slot outside a working-hour window or on a day already at the daily cap; a public caller can never override notice, host or horizon; public results contain free/busy only (no titles, attendees, sources or locations); all arithmetic in UTC with conversion through the IANA database (never fixed offsets for zones with daylight saving); a busy-sync never accepts an empty event list (the sync sweep deletes every future block of that source missing from the payload, so an empty payload after a failed fetch would silently wipe all future busy time); a busy-sync never touches blocks of another source or the owner's own bookings; nothing is written to any connected external calendar.
- **Soft constraints:** prefer slots inside `local_window`; spread suggestions across days; avoid offering the last slot before a long block; show times in the requester's zone first.
- **Resource and tenant scope:** one calendar owner's busy-store; sync writes limited to that owner's own sources.
- **Data sources allowed:** the busy-store; the owner's connected calendar (busy events only); invitation subject lines and dates from the owner's mailbox.
- **External transfers permitted:** none beyond returning the result to the caller; public callers receive free slots only.
- **Time limit:** 10 s per slot computation; whole run ≤ 60 s (≤ 3 minutes if a busy-sync refresh runs).
- **Cost or token budget:** no LLM required; calendar API calls ≤ 2 per refreshed source per run. Exact operator cap `TBD`.
- **Maximum steps:** 9 (S1–S9).
- **Maximum retries and correction cycles:** read steps 2 retries; busy-sync refresh 1 attempt per source per run; correction cycles 1.
- **Rate limits and concurrency:** public MCP callers limited per client (operator value `TBD`); one busy-sync per source at a time (a lock per source).
- **Retention and logging constraints:** slot results are not stored; logs hold counts and timings, never busy-block titles for public calls.
- **Prohibited actions:** booking, moving or cancelling anything; editing working hours or policy; deleting a busy block outside a full-window sync; revealing why a time is busy to anyone but the owner.
- **Stop conditions:** no working hours configured; busy-store unreachable; every busy source stale beyond the hard threshold (owner is told; public callers get `blocked`); request invalid after one clarification.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Busy block (one source) via `sync_calendar` / `sync_invites` | owner variant; source stale; non-empty full-window payload | Allowed (owner's own busy-store; no third party affected) | row count and window match the payload |
| Read | Working hours, policy, bookings (times only for public), busy blocks, sync status | own tenant only | Allowed | — |
| Update | Busy block `start_utc`, `end_utc`, `label`, `in_person` by (`source`, `ext_ref`) | same as Create | Allowed | upsert result matches the payload |
| Delete | Busy blocks of the same source absent from a full-window payload (cancelled or moved events) | only inside the sync sweep; refused when the payload is empty | Allowed (sweep only) | deleted count reported; zero deletions when the payload was refused |
| Delete | any booking, manual block or other source's block | — | Not Allowed | — |
| External side effect | MCP response to an external caller | public variant; free slots only | Allowed | response schema check; no busy detail present |

- **Idempotency:** slot computation is read-only. Busy-sync is idempotent by (`source`, `ext_ref`): replaying the same payload yields the same rows.
- **Expected side effects:** none for the slot computation; for a refresh, the owner's busy blocks for that source now mirror the source.
- **Compensation:** a wrong sync is corrected by re-running the sync with a correct full-window payload; the source of truth is the connected calendar or mailbox.
- **Irreversible:** nothing in the slot path. A sweep deletion is recoverable only by re-syncing from the source.
- **Ambiguous outcomes:** if a sync call times out, read `busy_sync.status` before retrying; never re-send a payload built from a fetch that has since failed.

## G. Autonomy and approval policy

- **Default autonomy level:** A2 — bounded autonomy (read-only computation).
- **Planning level:** A2.
- **Execution level:** A2; busy-sync refresh A2 within the owner's own sources.
- **Verification level:** A2.
- **Recovery level:** A2 — retry reads; degrade and disclose rather than invent.
- **Approval required for:** nothing in this recipe; any booking that follows is approved in recipe 22.
- **Actions never permitted:** creating or changing bookings; widening a public call to owner privileges; exposing busy detail to non-owners; deleting busy blocks outside a full-window sweep; writing to connected calendars.
- **Conditions requiring clarification:** ambiguous day or time zone; both `duration_min` and an unknown `event_type`; a range that is entirely outside working hours.
- **Conditions requiring escalation:** no working hours; all sources stale for more than 24 hours; repeated sync errors; working hours that contradict a request the owner insists on (for example a 07:00 meeting).
- **Safe state after stop, cancellation or escalation:** no writes in progress; no partial sync applied (the sync is transactional per source); the caller told what is missing.

## H. Execution plan

**S1 — Validate the request**
1. Objective: a schema-valid request with a concrete length and range.
2. Capability: none (local validation).
3. Input mapping: user message or MCP arguments → `request@0.1.0`.
4. Dependencies: none.
5. Preconditions: a scheduling question or a `get_slots` call.
6. Action / side effects: parse, normalise dates, resolve `event_type` to minutes later in S3; no side effects.
7. Autonomy: A2; no approval.
8. Timeout / retry: 2 s / none.
9. Idempotency: not applicable.
10. Postconditions: request valid; `to ≥ from`.
11. Verification: JSON Schema validation passes.
12. Failure routes: invalid → ask once (owner conversation) or return `blocked` with a validation error (MCP).
13. Escalation triggers: none.
14. Evidence: the validated request.

**S2 — Select the variant from the authenticated caller**
1. Objective: decide between `get_slots.default` and `get_slots.public`.
2. Capability: runtime identity (UNVERIFIED binding).
3. Input mapping: caller identity from the runtime, never from the request body.
4. Dependencies: S1.
5. Preconditions: the caller is authenticated (owner session, or an MCP bearer token for public use).
6. Action / side effects: choose the variant; for public, drop `min_notice_min` and any host or horizon override.
7. Autonomy: A2.
8. Timeout / retry: 2 s / none.
9. Idempotency: not applicable.
10. Postconditions: exactly one variant selected.
11. Verification: a public caller always maps to `public`; the stripped fields are absent from the outgoing call.
12. Failure routes: unauthenticated → `blocked` (permission denied).
13. Escalation triggers: none.
14. Evidence: variant ID and the list of stripped fields.

**S3 — Load working hours and policy**
1. Objective: know the owner's time zone, windows and rules.
2. Capability: `nova.calendar.availability.get_working_hours`.
3. Input mapping: owner ID.
4. Dependencies: S2.
5. Preconditions: variant selected.
6. Action / side effects: read-only.
7. Autonomy: A2.
8. Timeout / retry: 5 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: `working-hours@0.1.0` and `policy@0.1.0`; `event_type` resolved to minutes and buffer.
11. Verification: at least one window; every window has `end_min > start_min`; owner zone is a valid IANA name.
12. Failure routes: no windows → `blocked` and tell the owner to set working hours.
13. Escalation triggers: no working hours.
14. Evidence: policy values used (copied into `policy_applied`).

**S4 — Check busy-source freshness**
1. Objective: know whether the busy-store reflects reality.
2. Capability: `nova.calendar.busy_sync.status`.
3. Input mapping: owner ID.
4. Dependencies: S3.
5. Preconditions: none.
6. Action / side effects: read-only. A source is `fresh` if its last success is within 2 × its sync interval (60 minutes for a 30-minute schedule), `degraded` up to 24 hours, `stale` beyond, `never` if it has never synced. A success that returned zero items for a source that usually has items is marked `degraded`.
7. Autonomy: A2.
8. Timeout / retry: 5 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: a state per source and an overall state (worst of all sources).
11. Verification: every configured source has a state.
12. Failure routes: status unreadable → treat every source as `stale` and disclose.
13. Escalation triggers: all sources `stale` (owner variant) → tell the owner before returning slots.
14. Evidence: per-source states with timestamps.

**S5 — Refresh stale sources (owner variant only, optional)**
1. Objective: bring a stale or degraded source up to date before computing.
2. Capability: `nova.calendar.busy_sync.sync_calendar` and/or `nova.calendar.busy_sync.sync_invites`.
3. Input mapping: source ID; window now → now + horizon.
4. Dependencies: S4.
5. Preconditions: owner variant; source not `fresh`; no sync for that source already running.
6. Action / side effects: fetch the full window from the source; classify each event (skip all-day, cancelled, declined, "free" and non-default event types; in-person events get travel padding of 60 minutes before and after, decided negative-evidence-first — an event is online unless it shows physical-venue evidence, because video tools often put a URL or just a product name in the location field); skip the owner's own bookings by matching the stored external event ID; send one full-window payload. Busy blocks of that source are upserted and missing future ones deleted.
7. Autonomy: A2; no approval (owner's own busy-store).
8. Timeout / retry: 60 s per source / 1 attempt.
9. Idempotency: upsert by (`source`, `ext_ref`).
10. Postconditions: source `last_success_at` updated, or the source left untouched.
11. Verification: payload non-empty (else refused and nothing deleted); upserted + deleted counts reported; no block of another source changed.
12. Failure routes: fetch fails → do not sync; keep existing blocks; disclose `degraded`/`stale`. Empty payload → refused by the sync operation; disclose.
13. Escalation triggers: two consecutive refresh failures for the same source → tell the owner (the connection may need re-authorising).
14. Evidence: per-source sync result (counts, timestamps), not event titles.

**S6 — Compute slots**
1. Objective: candidate slots for the length and range.
2. Capability: `nova.calendar.availability.get_slots` (selected variant).
3. Input mapping: minutes (or event type), `from`, `to` converted to UTC at the owner's day boundaries, `min_notice_min` (owner only).
4. Dependencies: S3, S4 (and S5 if run).
5. Preconditions: policy loaded.
6. Action / side effects: read-only. The engine expands each working-hour window into UTC through the owner's zone (windows ending after 24:00 spill into the next day), steps by the granularity, keeps a start only if the whole length is free, removes starts before now + notice and after the horizon, removes overlaps with bookings expanded by the buffer and with every busy block (half-open overlap test), and skips days at the daily cap.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: zero or more slots in UTC.
11. Verification: each slot length = requested minutes; starts aligned to the granularity.
12. Failure routes: zero slots → widen nothing automatically; return an empty list with the reason (for example "fully booked until 9 Nov") and offer the next free day.
13. Escalation triggers: none.
14. Evidence: slot count, engine version, policy values.

**S7 — Verify slots independently**
1. Objective: prove no returned slot collides with anything.
2. Capability: `nova.calendar.availability.list_calendar`.
3. Input mapping: the earliest slot start minus the largest buffer → the latest slot end plus the largest buffer.
4. Dependencies: S6.
5. Preconditions: at least one slot.
6. Action / side effects: read every booking and busy block in that range and re-test each slot in code that is separate from the slot engine: no overlap (buffers applied to bookings), inside a working window converted independently, after now + notice, before the horizon.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: every slot passes, or failing slots are removed.
11. Verification: 100 % of returned slots pass all four tests.
12. Failure routes: any slot fails → remove it, record a `slot_engine_mismatch` defect and mark the run `partially_completed`; more than 20 % fail → return no slots, `failed`.
13. Escalation triggers: any mismatch is reported to the publisher as a defect (no busy detail in the report).
14. Evidence: pass/fail per slot (IDs and times only).

**S8 — Convert and annotate time zones**
1. Objective: human-correct times for both parties.
2. Capability: none (IANA conversion in the runtime).
3. Input mapping: verified UTC slots, `display_tz`, owner zone.
4. Dependencies: S7.
5. Preconditions: valid zones.
6. Action / side effects: render each slot in `display_tz` and the owner's zone with the abbreviation in force on that date; flag daylight-saving transition days and slots that cross midnight in either zone; add a note when the offset between the two zones changes inside the range; apply `local_window`; trim to `max_results`, spreading across days.
7. Autonomy: A2.
8. Timeout / retry: 2 s / none.
9. Idempotency: not applicable.
10. Postconditions: `slots[]` complete with display and owner strings.
11. Verification: converting `start_display` back to UTC equals `start_utc` for every slot.
12. Failure routes: round-trip mismatch → drop that slot and record a defect.
13. Escalation triggers: none.
14. Evidence: conversion check results.

**S9 — Return the result**
1. Objective: a truthful, schema-valid answer.
2. Capability: the conversation, or the MCP response for `get_slots`.
3. Input mapping: S6–S8 outputs, freshness from S4/S5.
4. Dependencies: S8.
5. Preconditions: result assembled.
6. Action / side effects: return `slot_result`; for public callers strip `freshness.sources` detail to a single `overall` value and remove anything but times.
7. Autonomy: A2.
8. Timeout / retry: 5 s / none.
9. Idempotency: not applicable.
10. Postconditions: caller has the result.
11. Verification: output validates against `slot-result@0.1.0`; public output contains no labels or sources.
12. Failure routes: schema failure → fix fields; never fabricate a slot or a freshness value.
13. Escalation triggers: none.
14. Evidence: the response (times only) and its status.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Input validity | request matches `request@0.1.0` | validator | valid | ask once, then `blocked` |
| Variant integrity | public caller → `public` variant; overrides stripped | S2 record vs outgoing call | exact | stop; `blocked`; report a policy defect |
| Working hours | at least one valid window | `get_working_hours` | valid | `blocked` |
| Freshness | every source has a state; empty-payload syncs refused | `busy_sync.status`, sync results | states present | disclose `degraded` / `stale` |
| No collision | each slot free of bookings + buffers and all blocks | `list_calendar` (independent of the engine) | 100 % pass | remove slot; defect; status downgrade |
| Policy bounds | each slot ≥ now + notice, ≤ horizon, inside a window, day under cap | independent re-check | 100 % pass | remove slot |
| Time-zone round trip | display → UTC equals `start_utc` | S8 check | equal | drop slot |
| Data boundary | public output has no titles, sources, attendees | output scan | none present | strip and record a defect |
| Output schema | `slot_result` valid | validator | valid | fix; never invent values |

A slot engine returning rows proves only that it ran; S7 is what shows the slots are actually free.

## J. Self-correction and recovery

- **Failure classes:** validation, missing working hours, unauthenticated caller, timeout, transient database or calendar-API error, stale source, empty sync payload, slot-engine mismatch, time-zone round-trip failure.
- **Permitted corrections:** ask once for a missing length or zone; retry reads (bounded); refresh a stale source once (owner variant); remove slots that fail S7 or S8.
- **Forbidden corrections:** widening a public call to owner privileges; overriding notice or horizon for a public caller; accepting an empty sync payload; deleting blocks to "make room"; presenting a stale result as fresh.
- **Maximum correction cycles:** 1 per step; 2 per run.
- **Reverification required after correction:** yes — a refreshed source re-runs S4; any change in the slot set re-runs S7 and S8.
- **Stop conditions:** correction budget exhausted; no working hours; busy-store unreachable.
- **Recovery or compensation:** a wrong sync is fixed by re-syncing the full window from the source.
- **What must be reported:** removed slots and why, any source left stale, any refresh that ran.

## K. Independent challenge

- **Assumptions to challenge:** that the requester's "Tuesday 3 pm" is in their zone, not the owner's; that a fixed offset is still correct after a daylight-saving change inside the range; that a sync "success" with zero events is real.
- **Alternative approach:** if very few slots exist, would a shorter meeting or a different week serve the request? Offer it rather than silently changing the length.
- **Contradictory evidence:** an invite block and a calendar block for the same meeting at different times (a reschedule seen by one source only) — keep both (safer) and disclose.
- **Could mislead:** an empty or stale source makes the owner look free; an in-person event misread as online loses its travel padding.
- **Separate verifier:** S7 re-tests every slot against raw intervals in code separate from the slot engine; S8 round-trips every conversion.
- **Findings requiring disclosure:** stale sources, removed slots, ambiguous time-zone readings, travel padding not applied because location evidence was missing.

## L. Human escalation

- **Triggers:** no working hours; all sources stale for more than 24 hours; two consecutive refresh failures; owner insists on a time outside working hours; repeated slot-engine mismatches.
- **Why the agent cannot decide:** changing working hours or re-authorising a calendar connection is the owner's choice; a mismatch means the engine cannot be trusted.
- **Information presented:** which source is stale and since when, the policy values in force, the slots removed by verification.
- **Decision requested:** set working hours; reconnect a source; accept a degraded result; book outside hours manually.
- **Options and consequences:** accepting a degraded result risks a double booking that recipe 22 will catch only at booking time.
- **Safe behaviour while waiting:** return nothing or a clearly labelled degraded list; no writes.
- **If no response:** after 30 minutes the run ends `blocked` (owner) or immediately `blocked` (public caller).
- **Resume:** S3 and S4 re-run before any slot is returned.

## M. Completion conditions

Mandatory: `slot_result` valid; every slot passed S7 and S8; public output free of busy detail; freshness reported truthfully; policy values reported; limitations disclosed.

- `completed` — slots returned (or a verified empty list with its reason) and every source `fresh`.
- `partially_completed` — slots returned but at least one source `degraded` or `stale`, or slots were removed by verification.
- `blocked` — invalid input after clarification, unauthenticated caller, no working hours, or busy-store unreachable.
- `failed` — the slot engine or verification failed beyond the correction budget (for example more than 20 % of slots failed S7).
- `cancelled` — the requester abandoned the request before slots were returned.

An empty slot list is `completed` only when every source is fresh and S7 confirms the range is truly busy.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome ("3 slots on Tue–Wed, shown in London time"); `slot_result`; step statuses S1–S9; verification results and failed checks; evidence references (policy snapshot, per-source freshness, verification summary); approvals (none required); side effects (busy-sync refresh counts, if any) and reconciliation status; corrections performed; unresolved challenges; limitations; escalation state; learning candidate. Never include connection secrets, busy-block titles for public callers, or attendee details.

## O. Learning and revision

- **Outcome metrics:** slots returned per request, empty-result rate, share of runs with stale sources, S7 mismatch rate, refresh success rate, later double-booking incidents traced to this recipe.
- **Evidence retained:** run record with counts, policy snapshot and freshness states, per the retention policy.
- **Data minimisation:** no event titles or attendee data in metrics; public-call logs keep times only.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}`.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review; no silent changes to trusted behaviour.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (sandbox calendar, fictional owner) | correct slots in the display zone; all pass S7/S8 | UNVERIFIED on Skill Port; observed in the reference implementation |
| Missing length | asks once, then `blocked` | TBD |
| Invalid zone ("EST" without mapping, unknown name) | rejected or asked | TBD |
| Range beyond horizon | clamped with a note | TBD |
| Missing capability (slot binding absent) | `blocked` with a dependency message | TBD |
| Permission denied (no `calendar:availability:read`) | `blocked`, no data returned | TBD |
| Unauthorized scope (public caller sends `min_notice_min: 0`) | override stripped; notice enforced | TBD |
| Tenant isolation | cannot read another owner's busy-store | TBD |
| Working hours crossing midnight (12:00–02:00) | post-midnight slots on the next calendar day | TBD |
| Buffer and adjacency | no slot inside a booking's buffer; a slot may start exactly at a block's end | TBD |
| Daily cap | no slots on a day at `max_bookings_per_day` | TBD |
| Daylight-saving transition in range | correct offsets before and after; flag and note present | TBD |
| Zero-length block in data | rejected by validation; reported | TBD |
| Empty sync payload | refused; existing blocks kept; source shown stale | TBD |
| Own booking in the connected calendar | skipped by external event ID; no double block | TBD |
| In-person event | blocked with 60 min travel either side; online events unpadded | TBD |
| Timeout on `get_slots` | retried up to 2 times, then `failed` | TBD |
| Duplicate / replay of a sync payload | identical rows; no duplicates | TBD |
| False success (engine returns a colliding slot) | S7 removes it; status downgraded; defect logged | TBD |
| Failed verification (> 20 % mismatched) | no slots, `failed` | TBD |
| Correction budget exhausted | stops safely with the reason | TBD |
| Escalation (all sources stale) | owner told before slots are returned | TBD |
| Cancellation (requester abandons) | `cancelled`, no writes | TBD |
| Rollback / compensation (bad sync corrected by re-sync) | blocks match the source again | TBD |
| Output schema and status | valid `slot_result`; public output has no busy detail | TBD |
| Compatibility / regression | passes after a time-zone database update | TBD |

All tests use a sandbox tenant, fictional events and rolled-back transactions; never the owner's real calendar or real guests. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, schema registry and MCP adapter registration — TBD (owner: publisher, with Skill Port).
- Public-caller rate limits and the per-client quota — TBD (owner: operator).
- Freshness thresholds are tied to a 30-minute sync schedule; other schedules need other thresholds.
- Invite parsing reads subject lines, whose format varies by calendar vendor and language; unknown formats are skipped, not guessed, so some invites may not block time.
- In-person detection is heuristic; travel padding is fixed (60 minutes) rather than computed from distance.
- Single host by design; multi-host and round-robin availability are not supported.
- For public calls, the S7 re-check reads busy detail server-side under the owner's installation, never with the caller's credentials; the platform must confirm it supports this delegation model (owner: publisher, with Skill Port).
- Cross-area IDs (mail metadata access) must be reconciled with the inbox recipes in this package.
- Reference-implementation evidence is historical; re-verify on the target runtime before relying on it.
