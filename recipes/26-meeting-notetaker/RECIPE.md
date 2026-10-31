# Meeting notetaker

## A. Identity and ownership

- **Recipe ID:** `nova.calendar.meeting-notetaker` (proposed until assigned by Skill Port)
- **Name:** Meeting notetaker
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** productivity
- **Tags:** meetings, notetaker, meeting-bot, recording, transcription, diarization, consent, minutes · Vedaxi task type: none — not one of the seven task types Vedaxi supports today
- **Summary:** On an explicit request, or for a calendar meeting the user opted in, a clearly named bot joins one online meeting from its link, announces that it is recording, records audio until the meeting ends, then produces a transcript with speaker labels and grounded minutes for the user. If consent is withdrawn, it leaves and discards everything.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** A dedicated bot account per meeting platform, provisioned by the deployer and signed in once by hand (the platform blocks automated sign-in; the browser profile persists); a headless browser runtime with a virtual audio sink and an audio encoder; a Whisper-class speech-to-text provider; an optional local speaker-diarization model; the minutes capabilities of `nova.calendar.meeting-minutes`; a private recording store; an owner notification channel; the calendar booking list (for auto-join). All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Unverified — catalogue status *Built, blocked*: deployment needs a dedicated bot account that the deployer provisions, and joining a meeting is unverified for this package. In Nova's reference implementation the downstream stages (audio compression and chunking, hosted Whisper-class transcription, local diarization, minutes, private recording storage and the owner recap) were observed working. Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.calendar.booking` / `.list` | Capability / Operation (read) | Read upcoming bookings with video links and attendees (MCP `upcoming_bookings` / `list_bookings`) | low | UNVERIFIED |
| `nova.notetaker.schedule` | Capability | Queue of opted-in meetings to auto-join | medium | UNVERIFIED |
| `nova.notetaker.schedule.add` | Operation (create) | Add one entry, keyed by calendar event ID | medium | UNVERIFIED |
| `nova.notetaker.schedule.claim_due` | Operation (update) | Atomically claim entries whose start time has arrived | medium | UNVERIFIED |
| `nova.notetaker.bot` | Capability | Run one meeting bot | high | UNVERIFIED |
| `nova.notetaker.bot.join` | Operation (create + external side effect) | Dispatch the bot to join, announce and record (async) | high | UNVERIFIED |
| `nova.notetaker.bot.live` | Operation (read) | List meetings the bot is in right now | low | UNVERIFIED |
| `nova.notetaker.bot.leave` | Operation (update + external side effect) | Make the bot leave now, with a reason | medium | UNVERIFIED |
| `nova.recordings.store` | Capability | Private recording storage | high | UNVERIFIED |
| `nova.recordings.store.upload` / `.delete` | Operations (create / delete) | Upload to a private bucket; delete on consent withdrawal or retention expiry | high | UNVERIFIED |
| `nova.transcribe.audio` | Capability | Speech-to-text and speaker turns | medium | UNVERIFIED |
| `nova.transcribe.audio.transcribe` / `.diarize` | Operations (read; external transfer for hosted) | Transcript segments; ephemeral speaker labels | medium | UNVERIFIED |
| `nova.minutes.extract.structured`, `nova.minutes.verify.grounding`, `nova.minutes.store.create` | Operations | Minutes, exactly as in `nova.calendar.meeting-minutes` | medium | UNVERIFIED |
| `nova.approvals.gateway.request` / `.status` | Operations (create / read) | Record and check the user's per-meeting approval or opt-in | medium | UNVERIFIED |
| `nova.notify.owner.send` | Operation (owner only) | Send the result to the requesting user | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.notetaker.bot.join.default` — headless browser automation using the persistent signed-in profile of the dedicated bot account; display name set per meeting; microphone and camera muted before joining; audio captured from a virtual audio sink; mode async. Binding `binding://nova/notetaker/join`. Only the default variant exists. There is no fallback to a managed third-party bot service: that would move audio to another processor and needs a new recipe version and consent review.
- `nova.transcribe.audio.transcribe.whisper-hosted` — selected first when the operator allows external transfer: compress to mono (64 kbit/s), and when the file exceeds the provider upload limit (about 25 MB in the reference) split into 15-minute chunks, transcribe each with its time offset and merge. About USD 0.04 per audio hour in the reference implementation.
- `nova.transcribe.audio.transcribe.whisper-local` — self-hosted; eligible when the operator forbids external transfer or the hosted provider is not ready. Slower on CPU; long meetings may exceed the time limit (disclosed). Hosted → local fallback is allowed (less exposure); local → hosted is never automatic, because it adds an external transfer.
- `nova.transcribe.audio.diarize.local-model` — ephemeral speaker turns from a local open-source model; each transcript segment takes the speaker whose turn overlaps it most. Skipped above 45 minutes of audio on CPU. No voiceprints and no recognising returning speakers across meetings.
- `nova.transcribe.audio.diarize.channel-split` — not eligible here (the bot captures one mixed stream); used by `nova.calendar.desktop-meeting-recorder`.
- `nova.transcribe.audio.diarize.none` — selected when audio exceeds 45 minutes on CPU, the model is not installed or the operator turns diarization off; minutes then carry `speaker_unlabelled`.
- All other operations: only the default variant. Bindings: `binding://nova/calendar/list-bookings`, `binding://nova/notetaker/schedule-add`, `binding://nova/notetaker/schedule-claim`, `binding://nova/notetaker/join`, `binding://nova/notetaker/live`, `binding://nova/notetaker/leave`, `binding://nova/recordings/upload`, `binding://nova/recordings/delete`, `binding://nova/transcribe/stt`, `binding://nova/transcribe/diarize`, `binding://nova/minutes/extract`, `binding://nova/minutes/verify`, `binding://nova/minutes/store-create`, `binding://nova/approvals/request`, `binding://nova/approvals/status`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: a containerised bot service (headless browser, virtual display and audio sink, encoder) behind an authenticated internal endpoint; a scheduler on the automation hub claims due queue entries every minute; recordings sit in a private bucket and play back only through a session-checked relay.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Schedule entry | `schema://nova/meeting-notetaker/schedule-entry@0.1.0` | the requesting user's tenant | create, update (claim, status) |
| Bot session | `schema://nova/meeting-notetaker/bot-session@0.1.0` | the requesting user's tenant | create, read, update (leave) |
| Recording | `schema://nova/meeting-notetaker/recording@0.1.0` | linked to one bot session | create, delete (consent withdrawal, retention) |
| Transcript segment | `schema://nova/meeting-notetaker/transcript-segment@0.1.0` | linked to one session | create |
| Consent record | `schema://nova/meeting-notetaker/consent-record@0.1.0` | linked to one session | create |
| Minutes session | `schema://nova/meeting-minutes/minutes@0.1.0` | the requesting user's tenant | create |

**Authorization requirements (declared, never a grant):** `calendar:bookings:read`, `notetaker:schedule:write`, `notetaker:join`, `notetaker:read`, `notetaker:leave`, `recordings:write`, `recordings:delete`, `transcribe:audio`, `minutes:create`, `notify:owner`. The signed-in bot profile is a credential: `secret_ref://notetaker-bot-profile` (encrypted at rest, never exported). Service and provider keys: `secret_ref://notetaker-service`, `secret_ref://stt`, `secret_ref://llm`.

**Policy / trust:** risk class `high` (recording people). Requires the S3 checkpoint, disclosure on by default, a meeting-link host allowlist, proof that the user organises or is invited to the meeting, one bot per meeting, and the deployer's review of each platform's terms on automated or business-function accounts.

## C. Intent and applicability

- **User problem:** The user wants notes from an online meeting without taking them by hand or paying a third-party notetaker service.
- **Desired outcome:** The bot joins only the intended meeting, everyone is told it is recording, and the user gets grounded minutes, a transcript with speaker labels where possible and a private recording reference.
- **Trigger or entry conditions:** The user says "join <link> and take notes" in chat or the portal, or a calendar meeting the user opted in reaches its start time.
- **Applicable situations:** Meetings the user organises or is invited to, on a web meeting platform that has a join adapter (the reference targets Google Meet; Zoom and Microsoft Teams adapters are TBD), up to 3 hours.
- **Out-of-scope situations:** meetings the user is not part of; joining silently or under a misleading name; phone-only or in-person meetings (see `nova.calendar.desktop-meeting-recorder`); speaking in the meeting; capturing video, screen shares or chat history; live notes; sharing minutes with attendees (a separate, approval-gated recipe); voiceprints; sentiment or emotion scoring; meetings where the operator's consent policy forbids bot recording.
- **Required inputs:** `meeting_url`, `trigger` (see Section D).
- **Prerequisites:** dedicated bot account provisioned and signed in; platform terms reviewed by the deployer; virtual audio sink healthy; speech-to-text ready; minutes capabilities installed; recording retention set.
- **Required capabilities:** `nova.notetaker.bot`, `nova.notetaker.schedule`, `nova.calendar.booking`, `nova.transcribe.audio`, `nova.recordings.store`, `nova.minutes.extract`, `nova.minutes.verify`, `nova.minutes.store`, `nova.approvals.gateway`, `nova.notify.owner` (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** the dedicated bot account is a deployer-provisioned dependency; until it exists every run is `blocked`. Join adapters beyond the reference platform: TBD.
- **Assumptions:** Observed in the reference implementation: transcription with chunking, diarization, minutes, private storage and recap. Unverified: joining for this package; admission behaviour (platforms may hold unrecognised participants for manual host admission; enterprise tenants may label or block third-party bots); meeting web UI selectors, which change without notice.
- **Ambiguity policy:** Ask. If the link, the meeting ("my 3 pm" with two matches) or the time is unclear, ask once; never join a meeting the user did not name.
- **Vedaxi task type:** none — not one of the seven supported today.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `meeting_url` | `https://` URL whose host is on the operator's allowlist | yes | user / calendar; confidential (a join link grants entry) | — | Not https or host not allowlisted: reject. |
| `trigger` | enum `on_demand`, `calendar` | yes | entry point | — | — |
| `title` | string, 3–200 chars | no | user / calendar | "Meeting on <date>" | — |
| `scheduled_start`, `calendar_event_id` | date-time; string | when `trigger = calendar` | calendar | — | Missing: entry rejected. Start moved: entry updated by event ID. |
| `consent_basis` | enum `all_party_announced`, `organizer_attested` | no | runtime policy | `all_party_announced` | Basis the policy does not accept for the jurisdiction: `blocked`. |
| `disclosure` | boolean | no | runtime policy | `true` | `false` only where operator policy and law allow; otherwise policy wins. |
| `bot_display_name` | string 3–60 chars, must contain "Notetaker" | no | runtime | "Notetaker — recording for <user's name>" | Missing the word: rejected. |
| `max_duration_s` | integer 300–10800 | no | user | 10800 | Out of range: clamp and tell the user. |
| `admission_timeout_s` | integer 60–1800 | no | user | 600 | Out of range: clamp. |
| `language` | ISO 639-1 code or `auto` | no | user | `auto` | Unsupported: stop and list supported languages. |

```json
{"$id":"schema://nova/meeting-notetaker/request@0.1.0","type":"object","required":["meeting_url","trigger"],
 "properties":{"meeting_url":{"type":"string","format":"uri","pattern":"^https://"},"trigger":{"enum":["on_demand","calendar"]},
 "title":{"type":"string","minLength":3,"maxLength":200},"scheduled_start":{"type":"string","format":"date-time"},
 "calendar_event_id":{"type":"string"},"consent_basis":{"enum":["all_party_announced","organizer_attested"]},
 "disclosure":{"type":"boolean"},"bot_display_name":{"type":"string","minLength":3,"maxLength":60,"pattern":"Notetaker"},
 "max_duration_s":{"type":"integer","minimum":300,"maximum":10800},"admission_timeout_s":{"type":"integer","minimum":60,"maximum":1800},
 "language":{"type":"string"}},
 "if":{"properties":{"trigger":{"const":"calendar"}}},"then":{"required":["scheduled_start","calendar_event_id"]}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `notetaker_result` | object `schema://nova/meeting-notetaker/result@0.1.0` | `session_id`, `status`, `joined`, `leave_reason`, `audio_health` | the requesting user's channel and the run record | confidential; owner only | cross-checked against the bot session, recording and minutes (Section I) |

```json
{"$id":"schema://nova/meeting-notetaker/result@0.1.0","type":"object",
 "required":["session_id","status","joined","leave_reason","audio_health"],
 "properties":{"session_id":{"type":"string"},"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},
 "joined":{"type":"boolean"},"admitted_at":{"type":["string","null"],"format":"date-time"},"left_at":{"type":["string","null"],"format":"date-time"},
 "leave_reason":{"enum":["meeting_ended","alone","user_leave","max_duration","admission_timeout","removed_by_host","consent_withdrawn","audio_failure","not_joined","error"]},
 "audio_health":{"enum":["ok","degraded","silent","not_captured"]},"duration_s":{"type":"integer","minimum":0},
 "recording_ref":{"type":["string","null"]},"transcript_ref":{"type":["string","null"]},"speakers_labelled":{"type":"boolean"},
 "minutes_ref":{"type":["string","null"]},"flags":{"type":"array","items":{"type":"string"}},
 "cost":{"type":"object","properties":{"amount":{"type":"number"},"currency":{"type":"string"},"breakdown":{"type":"object"}}}}}
```

Other schemas (compact). Booking, booking-ref, transcript, extraction and minutes schemas are reused from recipe 25 (`schema://nova/meeting-minutes/…@0.1.0`):

```json
{"$id":"schema://nova/meeting-notetaker/schedule-entry@0.1.0","type":"object","required":["entry_id","meeting_url","scheduled_start","status"],"properties":{"entry_id":{"type":"string"},"meeting_url":{"type":"string","format":"uri"},"title":{"type":"string"},"scheduled_start":{"type":"string","format":"date-time"},"source_ref":{"type":["string","null"],"description":"calendar event ID; unique"},"approval_id":{"type":["string","null"]},"status":{"enum":["pending","claimed","joined","done","cancelled","failed"]}}}
{"$id":"schema://nova/meeting-notetaker/bot-session@0.1.0","type":"object","required":["session_id","meeting_url","started_at","state"],"properties":{"session_id":{"type":"string"},"meeting_url":{"type":"string","format":"uri"},"title":{"type":"string"},"started_at":{"type":"string","format":"date-time"},"state":{"enum":["joining","waiting_admission","recording","leaving","ended"]}}}
{"$id":"schema://nova/meeting-notetaker/recording@0.1.0","type":"object","required":["recording_ref","session_id","duration_s"],"properties":{"recording_ref":{"type":"string"},"session_id":{"type":"string"},"bytes":{"type":"integer","minimum":0},"codec":{"type":"string"},"duration_s":{"type":"integer","minimum":0},"uploaded_at":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/meeting-notetaker/transcript-segment@0.1.0","type":"object","required":["seq","t_start_s","t_end_s","text"],"properties":{"seq":{"type":"integer","minimum":0},"t_start_s":{"type":"number","minimum":0},"t_end_s":{"type":"number","minimum":0},"speaker_label":{"type":["string","null"]},"text":{"type":"string"}}}
{"$id":"schema://nova/meeting-notetaker/consent-record@0.1.0","type":"object","required":["session_id","basis","withdrawn"],"properties":{"session_id":{"type":"string"},"basis":{"enum":["all_party_announced","organizer_attested"]},"announced_at":{"type":["string","null"],"format":"date-time"},"announcement_hash":{"type":"string"},"withdrawn":{"type":"boolean"},"withdrawn_reason":{"type":["string","null"]}}}
{"$id":"schema://nova/meeting-notetaker/session-ref@0.1.0","type":"object","required":["session_id"],"properties":{"session_id":{"type":"string"}}}
{"$id":"schema://nova/meeting-notetaker/leave-request@0.1.0","type":"object","required":["session_id","reason"],"properties":{"session_id":{"type":"string"},"reason":{"enum":["user_leave","consent_withdrawn"]}}}
{"$id":"schema://nova/meeting-notetaker/audio-ref@0.1.0","type":"object","required":["session_id","duration_s"],"properties":{"session_id":{"type":"string"},"recording_ref":{"type":["string","null"]},"local_ref":{"type":["string","null"]},"duration_s":{"type":"integer","minimum":0}}}
{"$id":"schema://nova/meeting-notetaker/approval@0.1.0","type":"object","required":["approval_id","scope_hash","decision"],"properties":{"approval_id":{"type":"string"},"scope_hash":{"type":"string"},"decision":{"enum":["approved","declined","expired","revoked"]},"decided_at":{"type":["string","null"],"format":"date-time"}}}
{"$id":"schema://nova/meeting-notetaker/claim-request@0.1.0","type":"object","required":["now","limit"],"properties":{"now":{"type":"string","format":"date-time"},"limit":{"type":"integer","minimum":1,"maximum":20}}}
{"$id":"schema://nova/meeting-notetaker/delivery@0.1.0","type":"object","required":["message_id","delivered_at"],"properties":{"message_id":{"type":"string"},"delivered_at":{"type":"string","format":"date-time"}}}
```

**Example (fictional)**

Input: `{"meeting_url":"https://meet.example.com/abc-defg-hij","trigger":"on_demand","title":"Vendor sync with John Roe and Ana Lima"}`

Output: `{"session_id":"ntk_0001","status":"completed","joined":true,"admitted_at":"2026-03-12T15:01:10-04:00","left_at":"2026-03-12T15:42:30-04:00","leave_reason":"meeting_ended","audio_health":"ok","duration_s":2480,"recording_ref":"run://ntk_0001/recording","transcript_ref":"run://ntk_0001/transcript","speakers_labelled":true,"minutes_ref":"min_0001","flags":[],"cost":{"amount":0.03,"currency":"USD","breakdown":{"stt":0.028,"llm":0.004}}}`

## E. Constraints and budgets

- **Hard constraints:** explicit request or a per-meeting opt-in; the user organises or is invited to the meeting; link host allowlisted; one bot per meeting; the visible name contains "Notetaker" and the user's name; microphone and camera stay muted and the bot never speaks; when disclosure is on (default) an announcement is posted in meeting chat within 30 s of admission; no capture before admission; audio only; at most 3 hours; consent withdrawn → leave, discard audio, do not transcribe; never train on audio; no voiceprints or cross-meeting speaker recognition; no sentiment or emotion scoring and no generated judgement about a person; the local audio file is deleted only after a confirmed upload or a discard; recordings are private.
- **Soft constraints:** leave promptly when alone; prefer hosted transcription for speed where the operator allows it.
- **Resource and tenant scope:** the requesting user's meetings, queue, sessions and recordings only.
- **Data sources allowed:** meeting audio between admission and leaving; the user's calendar bookings; meeting chat only to post the announcement.
- **External transfers permitted:** audio to the hosted speech-to-text provider (only if the operator enabled it); transcript text to the LLM provider; both configured not to retain or train.
- **Time limit:** admission ≤ `admission_timeout_s` (600 s); meeting ≤ `max_duration_s` (3 h); post-processing ≤ 30 minutes after leaving with hosted transcription (local: TBD).
- **Cost or token budget:** about USD 0.04 per audio hour for hosted transcription in the reference, plus minutes LLM tokens; per-run cap `TBD` (operator).
- **Maximum steps:** 11 (S1–S11).
- **Maximum retries and correction cycles:** join dispatch: 1 retry only after `live` confirms no session exists; never re-join after removal; transcription chunk: 2 retries; upload: 3 retries; correction cycles: 1 per step, 3 per run.
- **Rate limits and concurrency:** one bot per meeting; concurrent bots per tenant `TBD` (the reference ran one bot service).
- **Retention and logging constraints:** recordings kept per operator policy (`TBD`; recommend short, for example 7 days or less); logs hold IDs, timings and audio-level statistics, never audio or transcript text.
- **Prohibited actions:** joining any other meeting; re-joining after removal; unmuting; posting anything except the announcement; reading or storing chat history; keeping audio after consent withdrawal.
- **Stop conditions:** bot account missing or signed out; consent withdrawn; admission timeout; removal by the host; audio silent after the health check; budget exceeded.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Schedule entry (one per calendar event ID) | user opted this meeting in | Approval Required (opt-in) | entry readable; `source_ref` unique |
| Create | Bot session | S3 passed | Approval Required (S3) | `live` lists the session |
| Create | Recording (private) | admitted; consent not withdrawn | Allowed within the approved session | object exists; size and duration match |
| Create | Transcript, consent record, minutes session | recording kept | Allowed | schema-valid; read-back |
| Read | bookings, queue, live sessions | own tenant only | Allowed | — |
| Update | Schedule entry `status` (claim, mark); bot session `state` (leave) | atomic claim; session live | Allowed | status transitions logged |
| Delete | Recording and local audio | consent withdrawn (mandatory) or retention expiry | Allowed only for these two causes | object absent on read |
| Delete | anything else | — | Not Allowed | — |
| External side effect | bot visible in the meeting + chat announcement | S3 approval valid | Approval Required (S3) | admission and announcement logged |
| External side effect | audio to hosted STT; transcript to LLM | operator enabled transfer | Allowed (declared at installation) | provider request IDs |
| External side effect | result message to the requesting user | run ended | Allowed | delivery receipt |

- **Idempotency:** schedule entry key = calendar event ID, enforced by a full unique index on `source_ref` (a partial index cannot back an upsert conflict target); `claim_due` moves `pending → claimed` in one atomic statement with row locking, so two schedulers never dispatch the same entry; join key = hash(`tenant_id`, `meeting_url`, `scheduled_start` or `request_id`), and a duplicate returns the live session; upload key = `session_id`; minutes key = hash(`tenant_id`, `notetaker`, `session_id`).
- **Expected side effects:** a participant named "Notetaker — recording for Jane Doe" appears; one chat announcement; a private recording; provider charges.
- **Compensation:** `leave` ends presence immediately; on consent withdrawal, delete the recording, the local audio and any partial transcript.
- **Irreversible:** the bot having been seen; the announcement; audio already sent to a hosted provider; charges.
- **Ambiguous outcomes:** join timeout → read `live` by join key before any retry; upload timeout → check the object by key before re-uploading; never delete local audio while the upload state is unknown.

## G. Autonomy and approval policy

- **Default autonomy level:** A1 — supervised execution.
- **Planning level:** A2 (validate, resolve meeting, policy checks).
- **Execution level:** A1 — joins only after S3; recording runs within the stop rules.
- **Verification level:** A2.
- **Recovery level:** A1 — no re-join without a new approval.
- **Approval required for:** each on-demand join (link, title, display name, disclosure, duration cap); opting a calendar meeting in for auto-join (per meeting, or a revocable standing scope such as "meetings I organise"); disabling disclosure (operator policy plus user, only where law allows); keeping a recording beyond default retention.
- **Actions never permitted:** silent capture; a name that hides that it records; joining meetings the user is not part of; re-joining after removal; capture before admission; voiceprints; training on audio; sentiment or emotion scoring.
- **Conditions requiring clarification:** unclear link; more than one matching meeting.
- **Conditions requiring escalation:** admission timeout; removal by the host; a participant objects; audio health failure; bot account signed out or challenged by a platform security check; the platform labels or blocks the bot.
- **Safe state after stop, cancellation or escalation:** bot not in the meeting; nothing captured; audio discarded if consent was withdrawn, otherwise held privately; the user told.

Approval covers one meeting (or the stated standing scope). It does not cover other meetings or re-joins.

## H. Execution plan

**S1 — Validate the request**
1. Objective: a valid, allowlisted request.
2. Capability: none (local validation).
3. Input mapping: user message or queue entry → `request@0.1.0`.
4. Dependencies: none.
5. Preconditions: explicit request or opted-in entry.
6. Action / side effects: parse and validate; no side effects.
7. Autonomy: A2; no approval.
8. Timeout / retry: 5 s / none.
9. Idempotency: not applicable.
10. Postconditions: schema-valid request; host on the allowlist.
11. Verification: JSON Schema passes; host matches the allowlist exactly.
12. Failure routes: invalid or unallowlisted link → reject; missing link → ask once → `blocked`.
13. Escalation triggers: none.
14. Evidence: validated request (link masked in logs).

**S2 — Resolve the meeting and trigger**
1. Objective: know exactly which meeting, and that the user belongs to it.
2. Capability: `nova.calendar.booking.list`; for calendar entries `nova.notetaker.schedule.claim_due` (entries were added earlier by `nova.notetaker.schedule.add` at opt-in).
3. Input mapping: `meeting_url`, `calendar_event_id`, user identity.
4. Dependencies: S1.
5. Preconditions: request valid.
6. Action: on-demand — match the link to a booking where the user is organiser or invitee, or ask the user to confirm they are invited; calendar — atomically claim the due entry.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries on reads; the claim is not retried blindly (re-read status first).
9. Idempotency: claim is atomic; event ID unique.
10. Postconditions: one meeting identified; participation confirmed; entry `claimed` (calendar).
11. Verification: exactly one match; the entry was claimed by this run.
12. Failure routes: user not a participant → `blocked`; two matches → ask; already claimed elsewhere → stop silently (no duplicate).
13. Escalation: none.
14. Evidence: booking ID or user confirmation; claim result.

**S3 — Consent, policy and approval checkpoint**
1. Objective: confirm the join is allowed and approved now.
2. Capability: runtime policy (UNVERIFIED binding); `nova.approvals.gateway.request` / `.status` for calendar opt-ins.
3. Input mapping: link host, consent basis, disclosure, display name, duration cap, bot-account readiness, concurrency.
4. Dependencies: S2.
5. Preconditions: meeting resolved.
6. Action: check bot account signed in; consent basis accepted by policy; disclosure on unless policy permits off; no other bot in this meeting; then ask "Join 'Vendor sync' as 'Notetaker — recording for Jane Doe', announce recording, stop at 3 h?" (Approve / Edit / Cancel). For calendar entries, re-validate the stored opt-in (not revoked, meeting unchanged).
7. Autonomy: A0 for the decision.
8. Timeout / retry: on-demand approval waits 10 minutes, then `cancelled`.
9. Idempotency: the approval ID binds one meeting.
10. Postconditions: approval record with scope and timestamp; consent record started.
11. Verification: approval hash covers link, name, disclosure and cap.
12. Failure routes: bot account missing or signed out → `blocked`; policy fails → `blocked`; Cancel → `cancelled`.
13. Escalation: bot account challenged by a security check → ask the deployer to sign in again by hand.
14. Evidence: policy results, approval ID.

**S4 — Dispatch the bot**
1. Objective: start exactly one bot session.
2. Capability: `nova.notetaker.bot.join.default`.
3. Input mapping: link, title, display name, disclosure, `max_duration_s`, `admission_timeout_s`, join key.
4. Dependencies: S3.
5. Preconditions: approval valid; no live session for this meeting.
6. Action / side effects: the bot opens the link, mutes microphone and camera, sets the name and asks to join.
7. Autonomy: A1, covered by S3.
8. Timeout / retry: 30 s for dispatch; 1 retry only after `live` shows no session.
9. Idempotency: join key; duplicates return the existing session.
10. Postconditions: bot session in `joining` or `waiting_admission`.
11. Verification: `nova.notetaker.bot.live` lists the session with the approved link.
12. Failure routes: dispatch timeout → `live` by key; service rejects → `failed` with reason.
13. Escalation: platform shows a blocked or "unverified participant" banner → leave, `blocked`, report.
14. Evidence: session ID, dispatch timestamps.

**S5 — Admission and announcement**
1. Objective: be admitted, then tell everyone.
2. Capability: the `nova.notetaker.bot.join` session.
3. Input mapping: session ID, announcement template.
4. Dependencies: S4.
5. Preconditions: waiting for admission.
6. Action: wait for host admission (no capture while waiting); once admitted, post in chat: "This meeting is being recorded and transcribed by an AI notetaker for Jane Doe. Ask the host to remove me if you do not agree." Capture starts after the post succeeds.
7. Autonomy: A1.
8. Timeout / retry: `admission_timeout_s` (600 s); announcement 2 retries within 30 s.
9. Idempotency: one announcement per session.
10. Postconditions: `admitted_at` set; `announced_at` set when disclosure is on.
11. Verification: the bot sees itself in the participant list; the chat shows the announcement.
12. Failure routes: not admitted in time → leave, `leave_reason: admission_timeout`; announcement cannot be posted while disclosure is on → leave without capturing, `blocked`.
13. Escalation: admission timeout → tell the user (the platform may queue unrecognised participants for manual admission).
14. Evidence: admission and announcement timestamps, announcement hash.

**S6 — Record and monitor**
1. Objective: capture clean audio and leave at the right moment.
2. Capability: the bot session; `nova.notetaker.bot.live`; `nova.notetaker.bot.leave`.
3. Input mapping: session ID, stop rules.
4. Dependencies: S5.
5. Preconditions: admitted; announced when required.
6. Action: record from the virtual sink; check audio level and sink state in the first 60 s and every 5 minutes; leave when the meeting ends, when the bot is alone (after a 60 s grace, confirmed on 3 consecutive checks), at `max_duration_s`, or on `leave` from the user.
7. Autonomy: A1.
8. Timeout / retry: hard cap `max_duration_s`; on a silent sink, restart capture once.
9. Idempotency: `leave` is idempotent.
10. Postconditions: session `ended` with `leave_reason` and `audio_health`.
11. Verification: recorded duration ≈ `left_at − admitted_at` (±5 %); audio health not `silent`.
12. Failure routes: still silent after restart → leave, `audio_failure`, keep nothing claimed as a recording; removed by host → S7.
13. Escalation: silent audio or removal → tell the user.
14. Evidence: state transitions, audio-level summaries (no audio in logs).

**S7 — Consent-withdrawal branch**
1. Objective: honour an objection completely.
2. Capability: `nova.notetaker.bot.leave`; `nova.recordings.store.delete`.
3. Input mapping: session ID, reason (host removed the bot before or without consent, a participant objected, or the user pressed Leave with `consent_withdrawn`).
4. Dependencies: S5 or S6.
5. Preconditions: withdrawal signal received.
6. Action: leave at once; delete local audio and any uploaded object or partial transcript; mark the consent record `withdrawn`; skip S8–S10.
7. Autonomy: A2 — mandatory, no approval needed to discard.
8. Timeout / retry: leave within 5 s; delete 3 retries.
9. Idempotency: delete by `session_id`.
10. Postconditions: no audio or transcript remains.
11. Verification: local file absent; storage read returns not-found.
12. Failure routes: delete fails after retries → escalate to the operator; never transcribe meanwhile.
13. Escalation: as above.
14. Evidence: deletion confirmations, consent record.

**S8 — Store the recording**
1. Objective: one private, playable recording.
2. Capability: `nova.recordings.store.upload.default`.
3. Input mapping: compressed mono audio, `session_id`.
4. Dependencies: S6.
5. Preconditions: audio health not `silent`; consent not withdrawn.
6. Action / side effects: upload to the private bucket; playback only through an auth-checked relay; delete the local file only after the object is confirmed.
7. Autonomy: A2.
8. Timeout / retry: 120 s / 3 retries, each after checking the object by key.
9. Idempotency: key = `session_id`.
10. Postconditions: `recording_ref` set; no public URL exists.
11. Verification: object size and duration match; anonymous access is refused.
12. Failure routes: upload fails → keep the local file for recovery, continue to S9 from it, run `partially_completed`.
13. Escalation: none.
14. Evidence: `recording_ref`, size, checksum.

**S9 — Transcribe and diarize**
1. Objective: a transcript with timestamps and, where possible, speaker labels.
2. Capability: `nova.transcribe.audio.transcribe` (`whisper-hosted` → `whisper-local`); `nova.transcribe.audio.diarize` (`local-model` or `none`).
3. Input mapping: audio, `language`.
4. Dependencies: S8.
5. Preconditions: audio available.
6. Action: compress to mono; split into 15-minute chunks with offsets when over the upload limit; transcribe and merge; diarize unless audio exceeds 45 minutes on CPU; tag each segment with the most-overlapping speaker turn; discard speaker embeddings afterwards.
7. Autonomy: A2.
8. Timeout / retry: 5 minutes per chunk / 2 retries.
9. Idempotency: chunks keyed by `session_id` + offset.
10. Postconditions: ordered `transcript-segment@0.1.0` list.
11. Verification: segments cover ≥ 95 % of the audio timeline; no chunk missing; offsets monotonic.
12. Failure routes: a chunk keeps failing → mark the gap `[untranscribed mm:ss–mm:ss]`, run `partially_completed`.
13. Escalation: none.
14. Evidence: variant IDs, chunk count, coverage ratio, cost.

**S10 — Produce and verify minutes**
1. Objective: grounded minutes for the user.
2. Capability: `nova.minutes.extract.structured`, `nova.minutes.verify.grounding`, `nova.minutes.store.create` (recipe `nova.calendar.meeting-minutes`, steps S3–S9).
3. Input mapping: transcript segments, title, attendees, `meeting_start = admitted_at`, `source: notetaker`, `source_ref = session_id`, `recording_ref`, `consent: recorded`.
4. Dependencies: S9.
5. Preconditions: transcript non-empty.
6. Action: run the minutes flow unchanged; `speaker_unlabelled` when diarization was skipped.
7. Autonomy: A2.
8. Timeout / retry: as in recipe 25.
9. Idempotency: minutes key from Section F.
10. Postconditions: `minutes_ref` set.
11. Verification: recipe 25 grounding passes or its status is disclosed.
12. Failure routes: minutes fail → transcript still stored; run `partially_completed`.
13. Escalation: as in recipe 25.
14. Evidence: `minutes_ref`, verification report ID.

**S11 — Deliver the result**
1. Objective: tell the user what happened.
2. Capability: `nova.notify.owner.send.default`.
3. Input mapping: `notetaker_result`, minutes summary, flags.
4. Dependencies: S10, or S5–S7 when the run ended early.
5. Preconditions: result assembled.
6. Action: send one message to the requesting user with the summary, flags, recording and minutes references; mark the queue entry `done` or `failed`.
7. Autonomy: A1.
8. Timeout / retry: 15 s / 2 retries.
9. Idempotency: message key = `run_id`.
10. Postconditions: delivered; queue entry closed.
11. Verification: delivery receipt; entry status final.
12. Failure routes: delivery fails → result kept in the run record; `partially_completed`.
13. Escalation: none.
14. Evidence: message ID.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Input validity | request valid; host allowlisted | validator | valid | reject |
| Participation | user organises or is invited | booking or confirmation | true | `blocked` |
| Approval scope | joined link, name and cap equal the approved ones | approval vs session | equal | leave, `failed` |
| Single bot | at most one live session per meeting | `live` | ≤ 1 | stop the duplicate |
| No early capture | capture starts after admission and announcement | session timeline | holds | discard, escalate |
| Disclosure | announcement posted when disclosure is on | chat log entry | present | leave, `blocked` |
| Audio health | level and sink state OK at 60 s and every 5 min | monitor | not silent | restart once, then `audio_failure` |
| Consent | no withdrawal signal, else everything deleted | consent record, storage | consistent | delete, escalate |
| Private storage | object exists; anonymous read refused | storage probe | both hold | fix before delivery |
| Coverage | transcript covers ≥ 95 % of audio | S9 | holds | mark gaps |
| Minutes | recipe 25 checks | verification report | pass or disclosed | per recipe 25 |
| Output schema | `notetaker_result` valid; status per Section M | validator | valid | fix fields; never fake values |

`join` returning success proves only that a bot was dispatched, not that it was admitted, announced or recorded anything.

## J. Self-correction and recovery

- **Failure classes:** validation, missing capability (no bot account), sign-in challenge, admission timeout, platform block, removal, silent audio, transient provider error, upload failure, chunk failure, permission denial.
- **Permitted corrections:** one dispatch retry after `live` confirms no session; one capture restart on a silent sink; chunk and upload retries; hosted → local transcription fallback.
- **Forbidden corrections:** re-joining after removal or objection; joining under another name; disabling the announcement to get admitted; capturing while waiting for admission; local → hosted fallback without the operator's transfer setting; reporting a silent file as a recording.
- **Maximum correction cycles:** 1 per step; 3 per run.
- **Reverification required after correction:** yes — re-run that step's checks.
- **Stop conditions:** correction budget exhausted; a hard constraint would be violated.
- **Recovery or compensation:** leave; on withdrawal delete everything; on failed upload keep the local file for recovery.
- **What must be reported:** leave reason, audio health, gaps, deletions, retries and charges.

## K. Independent challenge

- **Assumptions to challenge:** that admission means consent from everyone; that the meeting joined is the one intended; that a quiet file means a quiet meeting.
- **Alternative approach:** a recorder on the user's own computer (`nova.calendar.desktop-meeting-recorder`) or the platform's own transcript where the user's organisation provides one; suggest these when bots are blocked.
- **Contradictory evidence:** a participant count that drops right after the announcement; chat replies objecting; platform warnings about the bot.
- **Could mislead:** minutes from a partial recording presented as complete; speaker names guessed from voices.
- **Separate verifier:** coverage and duration checks run on the stored object, not on the bot's own report; minutes use recipe 25's independent verifier.
- **Findings requiring disclosure:** gaps, skipped diarization, late admission, any objection.

## L. Human escalation

- **Triggers:** admission timeout; removal or objection; silent audio; bot account signed out or challenged; platform block; deletion failure after withdrawal.
- **Why the agent cannot decide:** consent, platform access and legal exposure belong to the user or deployer.
- **Information presented:** meeting title, timeline, leave reason, audio health, what was kept or deleted.
- **Decision requested:** re-approve a fresh join (not after removal); sign the bot account in again; accept partial notes; use another capture route.
- **Options and consequences:** stated per trigger (for example, a fresh join after a timeout shows the bot again to everyone).
- **Safe behaviour while waiting:** bot out of the meeting; nothing captured.
- **If no response:** after 30 minutes the run ends with what it has (`partially_completed`, `blocked` or `cancelled`).
- **Resume:** a new S3 approval; policy checks re-run first.

## M. Completion conditions

Mandatory: `notetaker_result` valid; joined only the approved meeting; announcement posted when required; no capture outside admission; consent honoured; storage private; minutes grounded or status disclosed; the user told; evidence references present.

- `completed` — admitted, announced, recorded with healthy audio, stored, transcribed and minuted; user notified.
- `partially_completed` — gaps in audio or transcript, diarization skipped, upload or minutes failed but the transcript exists, or delivery failed.
- `blocked` — missing bot account, sign-in challenge, policy or participation check failed, platform block, admission timeout, announcement impossible, or consent withdrawn (audio discarded, reported as `consent_withdrawn`).
- `failed` — dispatch or capture failed, or audio stayed silent, after the correction budget.
- `cancelled` — the user declined at S3, approval timed out, or the user withdrew the request before admission.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome; `notetaker_result`; step statuses S1–S11; verification results and failed checks; evidence references (approval ID, session ID, recording, transcript and minutes references, consent record); approvals and scope; side effects (bot presence, announcement, uploads, provider charges) and reconciliation status; deletions; corrections performed; unresolved challenges; limitations; escalation state; learning candidate. Never include raw audio, full transcripts in shared channels, meeting links in logs, or credentials.

## O. Learning and revision

- **Outcome metrics:** admission rate and wait time, announcement success, audio-failure rate, coverage ratio, diarization-skip rate, cost per audio hour, withdrawal rate.
- **Evidence retained:** run record, consent record, approval record, per the retention policy.
- **Data minimisation:** metrics without audio or text; no speaker embeddings kept.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}`.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests, including a fresh sandbox-meeting join after any meeting-UI change.
- **Promotion approval:** publisher review plus Skill Port review; consent and disclosure behaviour can never change silently.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (sandbox meeting hosted by the tester, consenting test participants) | admitted, announced, recorded, minuted | UNVERIFIED — joining unverified for this package; downstream stages observed in the reference implementation |
| Invalid input (http link, unallowlisted host) | rejected | TBD |
| Missing input (no link) | asks once, then `blocked` | TBD |
| Ambiguous meeting ("my 3 pm", two matches) | asks which | TBD |
| Missing capability (no bot account) | `blocked` with a dependency message | TBD |
| Bot account signed out or challenged | `blocked`; deployer asked to sign in | TBD |
| Permission denied (`notetaker:join` missing) | `blocked`, no dispatch | TBD |
| Unauthorized scope (meeting the user is not part of) | `blocked` | TBD |
| Tenant isolation | cannot list another tenant's live sessions or recordings | TBD |
| Admission timeout | leaves; `admission_timeout` | TBD |
| Timeout on dispatch | `live` read by key; no second bot | TBD |
| Duplicate / replay (two schedulers; same event synced twice) | one entry, one claim, one bot | TBD |
| Silent audio (sink disabled) | detected within 60 s; restart; then `audio_failure` | TBD |
| Long audio (> 25 MB, > 45 min) | chunked transcription; diarization `none`; flag set | TBD |
| False success (dispatched but never admitted) | no recording; not `completed` | TBD |
| Failed verification (minutes unsupported) | recipe 25 correction path | TBD |
| Correction budget exhausted | stops safely | TBD |
| Escalation (removed by host) | leaves; no re-join; user told | TBD |
| Consent withdrawal (participant objects) | leaves; audio and transcript deleted; `consent_withdrawn` | TBD |
| Cancellation (user declines at S3; user presses Leave) | `cancelled` / leave within 5 s with captured audio kept | TBD |
| Compensation (delete on withdrawal) | storage and local file both absent | TBD |
| Platform block (tenant blocks third-party bots) | `blocked`, reported | TBD |
| Output schema and status | valid `notetaker_result`; status per Section M | TBD |
| Compatibility / regression (meeting UI update) | sandbox join still passes | TBD |

All tests use sandbox meetings hosted by the tester with consenting test participants; never real customers or real meetings. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Dedicated bot account provisioning and each platform's terms on automated or business-function accounts — owner: deployer; blocks activation.
- Join adapters for Zoom and Microsoft Teams — TBD (owner: publisher).
- Meeting web UIs change without notice (join and announcement selectors need a regression test after updates); platforms may hold unrecognised participants for manual admission, and enterprise tenants may label or block third-party bots by default.
- Consent law: all-party-consent jurisdictions exist and some impose statutory damages per participant; the runtime must follow local law and the deployer should get legal review.
- Recording retention default, per-tenant concurrency and local transcription speed on CPU for long meetings — TBD (owner: operator).
- Optional objection detection in chat (an opt-in phrase list, chat text never stored) — open design question.
