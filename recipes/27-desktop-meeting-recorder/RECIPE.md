# Desktop meeting recorder

## A. Identity and ownership

- **Recipe ID:** `nova.calendar.desktop-meeting-recorder` (proposed until assigned by Skill Port)
- **Name:** Desktop meeting recorder
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** productivity
- **Tags:** meetings, desktop-capture, transcription, minutes, consent, no-silent-capture, sleep-safe · Vedaxi task type: none (not one of the seven task types Vedaxi supports today)
- **Summary:** Only after the user explicitly presses Record in a paired desktop app, the app captures the user's microphone and the computer's meeting audio on two separate tracks behind a visible recording indicator, stops cleanly on Stop or when the computer sleeps, relays audio chunks transiently for transcription, and turns the transcript into structured, grounded minutes. It never starts recording on its own.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** A desktop operating system that lets a signed app capture other applications' audio with the user's permission (reference: macOS 14.4 or later, Core Audio process taps); a signed app build (the system-audio permission prompt is only offered to signed builds); a paired account on the user's Nova installation; a Whisper-class speech-to-text provider reached through an authenticated relay; the minutes pipeline (contracts shared with recipe 25, Meeting minutes); an owner notification channel. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Unverified — catalogue status "Built, blocked". The app and its capture sidecar are built (shell, UI and core compile cleanly) but capture has not been tested on the target computer; the sleep-safe stop behaviour is written and reviewed but not applied, pending tests T1–T6 on that computer. The server side (pairing with a confirm code, device-token relay, action allowlist, transcript ingest into minutes) was deployed and smoke-tested in Nova's reference implementation. Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.desktop.pairing` | Capability | Bind one desktop device to the user's account with a revocable device token | medium | UNVERIFIED |
| `nova.desktop.pairing.pair` | Operation (create) | Device shows a 6-digit code; the user confirms it in the portal; a device token is issued | medium | UNVERIFIED |
| `nova.desktop.pairing.revoke` | Operation (update; kill switch) | Revoke a device token immediately | low | UNVERIFIED |
| `nova.desktop.recorder` | Capability | Local capture on the user's own computer, started only by an explicit Record press | high | UNVERIFIED |
| `nova.desktop.recorder.start` | Operation (create, local) | Begin capture after a Record press and show the recording indicator | high | UNVERIFIED |
| `nova.desktop.recorder.stop` | Operation (update, local) | Finalise the current chunk and end capture (Stop, cap, sleep, permission loss) | low | UNVERIFIED |
| `nova.desktop.recorder.status` | Operation (read, local) | Return state, elapsed time, indicator state and per-track audio level | low | UNVERIFIED |
| `nova.desktop.stt_relay` | Capability | Relay one audio chunk to speech-to-text under the device token; no server-side audio storage | high | UNVERIFIED |
| `nova.desktop.stt_relay.transcribe_chunk` | Operation (external side effect) | Send one chunk through the relay and return timed text segments | high | UNVERIFIED |
| `nova.transcribe.audio` | Capability | Speech-to-text and speaker attribution | medium | UNVERIFIED |
| `nova.transcribe.audio.transcribe` | Operation (read, external provider) | Whisper-class transcription of one chunk | medium | UNVERIFIED |
| `nova.transcribe.audio.diarize` | Operation (read) | Attribute transcript segments to speakers | medium | UNVERIFIED |
| `nova.desktop.minutes_ingest` | Capability | Accept a finished device transcript into the minutes pipeline | medium | UNVERIFIED |
| `nova.desktop.minutes_ingest.submit` | Operation (create) | Post the merged transcript and metadata; returns a minutes session ID | medium | UNVERIFIED |
| `nova.minutes.extract` | Capability | Turn a transcript into structured minutes | low | UNVERIFIED |
| `nova.minutes.extract.structured` | Operation (read, LLM) | Summary points, decisions, action items with owners and due dates | low | UNVERIFIED |
| `nova.minutes.verify` | Capability | Check minutes against the transcript | low | UNVERIFIED |
| `nova.minutes.verify.grounding` | Operation (read, LLM plus rules) | Every item must cite transcript segments; unsupported items are flagged or removed | low | UNVERIFIED |
| `nova.minutes.store` | Capability | Persist and read minutes sessions | medium | UNVERIFIED |
| `nova.minutes.store.create` | Operation (create) | Save one minutes session with transcript segments | medium | UNVERIFIED |
| `nova.minutes.store.get` | Operation (read) | Read one minutes session | low | UNVERIFIED |
| `nova.notify.owner` | Capability | Deliver a message to the requesting user's own channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Tell the user the minutes are ready | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.desktop.recorder.start.two-track` — default. Local IPC from the app UI to the capture sidecar; microphone track (the user) and system-audio track (other participants) are written separately with per-track start offsets. Binding `binding://nova/desktop/recorder-start` (UNVERIFIED), protocol: local IPC; mode: sync start, then continuous capture. No remote, scheduled, calendar-triggered, API-triggered or hidden start variant exists, and none may be added (design invariant, not a missing feature).
- `nova.desktop.recorder.start.mic-only` — eligible only when the user explicitly chooses it (an in-person meeting, or the system-audio permission is not granted and the user accepts the limitation). Never a silent fallback from two-track: it changes what is captured and removes speaker separation.
- `nova.transcribe.audio.transcribe.whisper-hosted` — selected. Hosted Whisper-class model reached only through `nova.desktop.stt_relay`; about USD 0.08 per meeting hour in the reference implementation (both tracks). `.whisper-local` is declared but not eligible in this version (on-device "private mode" is deferred in the reference). No silent switch in either direction, because cost, latency and accuracy differ.
- `nova.transcribe.audio.diarize.channel-split` — selected for two-track. Microphone segments are labelled "Me", system segments "Others", aligned by track offsets. Identifies both speakers on a 1:1 call; on group calls every remote speaker shares the "Others" label (disclosed).
- `nova.transcribe.audio.diarize.none` — selected for mic-only; segments are labelled "Room". `.local-model` is not eligible here: it needs the audio on a server or a local diarization model, and this recipe has neither. Declared for compatibility with recipe 26.
- `nova.minutes.extract.structured.single-pass` — transcripts up to a token limit (`TBD`, set by the configured model); `nova.minutes.extract.structured.chunked` — longer transcripts, extracted per section then merged into the same output schema. Selection is by transcript length only. All other operations: only the default variant is supported.

**Bindings:** `binding://nova/desktop/pair`, `binding://nova/desktop/revoke`, `binding://nova/desktop/recorder-start`, `binding://nova/desktop/recorder-stop`, `binding://nova/desktop/recorder-status`, `binding://nova/desktop/stt-relay`, `binding://nova/transcribe/stt`, `binding://nova/transcribe/diarize`, `binding://nova/desktop/minutes-ingest`, `binding://nova/minutes/extract`, `binding://nova/minutes/verify`, `binding://nova/minutes/store-create`, `binding://nova/minutes/store-get`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: a menu-bar desktop app (Tauri 2 shell, React/TypeScript UI, Rust core) with a Swift capture sidecar over local IPC; server routes behind authenticated HTTPS that accept a device token; minutes in a database exposed only through security-definer functions with row-level security.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Device pairing | `schema://nova/desktop-meeting-recorder/pairing@0.1.0` (the token itself is never in a schema; it lives in the OS keychain as `secret_ref://device-token`) | the user | create (pair), update (revoke) |
| Recording session (local) | `schema://nova/desktop-meeting-recorder/recording-session@0.1.0` | the user's computer | create, read, update (end fields only) |
| Audio chunk (local spool) | `schema://nova/desktop-meeting-recorder/chunk@0.1.0` | one recording | create, read, delete (local, after ingest or retention) |
| Transcript segment | `schema://nova/desktop-meeting-recorder/transcript-segment@0.1.0` | the user's tenant | create (via ingest), read |
| Minutes session | `schema://nova/desktop-meeting-recorder/minutes@0.1.0` (shared contract with recipe 25) | the user's tenant | create, read |

**Authorization requirements (declared, never a grant):** `device:pair` (user confirms the code while signed in), `device:revoke`, `stt:relay` and `minutes:ingest` (device token, allowlisted), `minutes:write` and `minutes:read` (server-side pipeline, user's tenant), `notify:owner`. Operating-system permissions (`os:microphone`, `os:system-audio`): each prompted to the user by the OS; the recorder operations run locally under those permissions (protocol `other` = local IPC in the manifest). The device token is accepted only for a small action allowlist (relay, ingest, status and a few reads); any other action returns `403 forbidden_for_device`. A revoked or invalid token returns `401`; an unreachable auth backend returns `503 auth_unavailable`, which the app must never treat as unpairing. STT credentials stay server-side as `secret_ref://stt`; the LLM key as `secret_ref://llm`.

**Policy / trust:** risk class `high` (recording people). Requires the no-silent-capture invariant, the consent reminder (default on), a signed build, transient-only server handling of audio, a bounded local spool, and no cross-meeting speaker recognition.

## C. Intent and applicability

- **User problem:** The user takes meetings in many apps on one computer and wants accurate notes without a bot joining the call or a per-platform integration.
- **Desired outcome:** A recording made only by the user's explicit action and with everyone's knowledge, turned into grounded minutes, with nothing captured silently and no audio kept on the server.
- **Trigger or entry conditions:** The user presses Record in the desktop app and confirms the consent reminder.
- **Applicable situations:** Online meetings in any app on the computer; 1:1 calls (best speaker separation); in-person meetings in mic-only mode.
- **Out-of-scope situations:** Recording anyone without their knowledge; automatic, scheduled, calendar-triggered or remote start; recording on phones (mobile operating systems do not let third-party apps capture other apps' call audio); recording a meeting the user is not part of; recognising speakers across meetings (voiceprints); sentiment or emotion scoring; sharing minutes with others (a separate recipe with approval); keeping audio archives.
- **Required inputs:** `record_press`, `consent_confirmed` (see Section D).
- **Prerequisites:** Signed app build installed; device paired; microphone and system-audio permission granted; supported OS version; free disk space for the local spool.
- **Required capabilities:** `nova.desktop.pairing`, `nova.desktop.recorder`, `nova.desktop.stt_relay`, `nova.transcribe.audio`, `nova.desktop.minutes_ingest`, `nova.minutes.extract`, `nova.minutes.verify`, `nova.minutes.store`, `nova.notify.owner` (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port; every binding is a dependency to resolve before activation. On-device transcription and diarization are declared but not eligible.
- **Assumptions:** Verified in the reference implementation (server side): confirm-code pairing, device-token relay, allowlist enforcement, `503`-versus-`401` handling, transcript ingest into minutes. Built but not tested on the target computer: two-track capture and chunk upload. Written but not applied: the sleep-safe stop. Assumed: the target runtime offers an equivalent STT provider; the user knows the recording law where each participant is.
- **Ambiguity policy:** Stop for anything that would start capture — never assume a start. For title and attendees use a documented safe default ("Untitled meeting", owners limited to "Me" and named attendees). Language defaults to auto-detect.
- **Vedaxi task type:** none of the seven supported today.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `record_press` | event `{source: ui_button \| ui_shortcut, pressed_at}` from the focused app UI | yes | the user, locally | — | Absent: nothing starts. Any other source (API, schedule, remote, script): rejected and logged. Stale (> 5 s before capture would begin) or spanning a sleep/wake: discarded; the user must press again. |
| `consent_confirmed` | boolean ("I have told everyone this is recorded") | yes while the reminder is on (default) | the user | `false` | `false` or absent: no capture; the reminder shows again. |
| `mode` | enum `two_track`, `mic_only` | no | the user | `two_track` | `two_track` without system-audio permission → `blocked` with instructions; never a silent switch to `mic_only`. |
| `title` | string, 1–120 chars | no | the user; may be personal | "Untitled meeting <local date>" | Too long: truncate and tell the user. |
| `attendees` | array of `{name, email?}`, ≤ 50 | no | the user; personal data | `[]` | Never invented; action-item owners are matched only against these and "Me". |
| `language` | BCP-47 code or `auto` | no | the user | `auto` | Unsupported: stop before recording and list supported languages. |
| `max_duration_min` | integer 5–240 | no | the user or operator | `TBD` (open product question) | Out of range: clamp and tell the user. |
| `spool_retention_days` | integer 1–7 | no | operator | 7 | Out of range: clamp. |

```json
{"$id":"schema://nova/desktop-meeting-recorder/request@0.1.0","type":"object","required":["record_press","consent_confirmed"],
 "properties":{"record_press":{"type":"object","required":["source","pressed_at"],"properties":{"source":{"enum":["ui_button","ui_shortcut"]},"pressed_at":{"type":"string","format":"date-time"}}},
 "consent_confirmed":{"type":"boolean"},"mode":{"enum":["two_track","mic_only"]},"title":{"type":"string","minLength":1,"maxLength":120},
 "attendees":{"type":"array","maxItems":50,"items":{"type":"object","required":["name"],"properties":{"name":{"type":"string"},"email":{"type":"string","format":"email"}}}},
 "language":{"type":"string"},"max_duration_min":{"type":"integer","minimum":5,"maximum":240},"spool_retention_days":{"type":"integer","minimum":1,"maximum":7}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `recording_result` | object `schema://nova/desktop-meeting-recorder/recording-result@0.1.0` | `recording_id`, `status`, `end_reason`, `chunks_total`, `chunks_transcribed`, `audio_stored_server_side` | the app, the user's channel and the run record | personal data; owner only | cross-checked against the local session, chunk receipts and the stored minutes (Section I) |

```json
{"$id":"schema://nova/desktop-meeting-recorder/recording-result@0.1.0","type":"object",
 "required":["recording_id","status","end_reason","chunks_total","chunks_transcribed","audio_stored_server_side"],
 "properties":{"recording_id":{"type":"string"},"minutes_id":{"type":["string","null"]},
 "status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},"end_reason":{"enum":["user_stopped","ended_by_sleep","max_duration","permission_revoked","app_quit","capture_error","not_started"]},
 "mode":{"enum":["two_track","mic_only"]},"duration_s":{"type":"integer","minimum":0},"chunks_total":{"type":"integer","minimum":0},"chunks_transcribed":{"type":"integer","minimum":0},
 "gaps":{"type":"array","items":{"type":"object","properties":{"start_ms":{"type":"integer"},"end_ms":{"type":"integer"},"reason":{"type":"string"}}}},"empty_tracks":{"type":"array","items":{"enum":["mic","system"]}},
 "summary_points":{"type":"array","items":{"type":"string"},"maxItems":8},"decisions":{"type":"array","items":{"type":"string"}},
 "action_items":{"type":"array","items":{"type":"object","required":["task","owner","confidence"],"properties":{"task":{"type":"string"},"owner":{"type":"string"},"due_date":{"type":["string","null"],"format":"date"},"confidence":{"enum":["high","medium","low"]}}}},
 "audio_stored_server_side":{"const":false},"cost":{"type":"object","properties":{"amount":{"type":"number"},"currency":{"type":"string"},"breakdown":{"type":"object"}}}}}
```

Other schemas (compact):

```json
{"$id":"schema://nova/desktop-meeting-recorder/pairing@0.1.0","type":"object","required":["device_id","status"],"properties":{"device_id":{"type":"string"},"code":{"type":"string","pattern":"^[0-9]{6}$"},"status":{"enum":["pending","confirmed","revoked","expired"]},"confirmed_at":{"type":["string","null"],"format":"date-time"}}}
{"$id":"schema://nova/desktop-meeting-recorder/recording-session@0.1.0","type":"object","required":["recording_id","device_id","mode","started_at","indicator_shown","tracks"],"properties":{"recording_id":{"type":"string"},"device_id":{"type":"string"},"mode":{"enum":["two_track","mic_only"]},"started_at":{"type":"string","format":"date-time"},"ended_at":{"type":["string","null"],"format":"date-time"},"end_reason":{"enum":["user_stopped","ended_by_sleep","max_duration","permission_revoked","app_quit","capture_error","not_started",null]},"indicator_shown":{"const":true},"tracks":{"type":"array","minItems":1,"items":{"type":"object","required":["track","start_offset_ms"],"properties":{"track":{"enum":["mic","system"]},"start_offset_ms":{"type":"integer","minimum":0}}}}}}
{"$id":"schema://nova/desktop-meeting-recorder/chunk@0.1.0","type":"object","required":["recording_id","track","seq","start_ms","duration_ms","sha256"],"properties":{"recording_id":{"type":"string"},"track":{"enum":["mic","system"]},"seq":{"type":"integer","minimum":0},"start_ms":{"type":"integer","minimum":0},"duration_ms":{"type":"integer","minimum":1},"sha256":{"type":"string","pattern":"^[a-f0-9]{64}$"},"uploaded":{"type":"boolean"}}}
{"$id":"schema://nova/desktop-meeting-recorder/transcript-segment@0.1.0","type":"object","required":["recording_id","track","speaker","start_ms","end_ms","text"],"properties":{"recording_id":{"type":"string"},"track":{"enum":["mic","system"]},"speaker":{"enum":["Me","Others","Room"]},"start_ms":{"type":"integer","minimum":0},"end_ms":{"type":"integer","minimum":0},"text":{"type":"string"}}}
{"$id":"schema://nova/desktop-meeting-recorder/minutes@0.1.0","type":"object","required":["minutes_id","title","source","summary_points","decisions","action_items"],"properties":{"minutes_id":{"type":"string"},"title":{"type":"string"},"source":{"const":"desktop"},"summary_points":{"type":"array","items":{"type":"string"}},"decisions":{"type":"array","items":{"type":"string"}},"action_items":{"type":"array","items":{"type":"object","required":["task","owner","confidence"],"properties":{"task":{"type":"string"},"owner":{"type":["string","null"]},"due_date":{"type":["string","null"],"format":"date"},"confidence":{"enum":["high","medium","low"]},"source_refs":{"type":"array","items":{"type":"string"}}}}},"low_confidence_flags":{"type":"array","items":{"type":"string"}}}}
{"$id":"schema://nova/desktop-meeting-recorder/delivery@0.1.0","type":"object","required":["message_id","delivered_at"],"properties":{"message_id":{"type":"string"},"delivered_at":{"type":"string","format":"date-time"},"channel":{"type":"string"}}}
```

**Example (fictional)**

Input: `{"record_press":{"source":"ui_button","pressed_at":"2026-11-03T15:00:02-05:00"},"consent_confirmed":true,"mode":"two_track","title":"Budget check-in","attendees":[{"name":"John Roe","email":"john@example.org"}],"language":"en"}`

Output: `{"recording_id":"rec_0001","minutes_id":"min_0001","status":"completed","end_reason":"user_stopped","mode":"two_track","duration_s":1820,"chunks_total":8,"chunks_transcribed":8,"gaps":[],"empty_tracks":[],"summary_points":["Q1 budget stays flat.","Travel line moves to the workshop."],"decisions":["Keep the Q1 budget flat."],"action_items":[{"task":"Send the revised budget sheet","owner":"John Roe","due_date":"2026-11-06","confidence":"high"},{"task":"Book the workshop room","owner":"Me","due_date":null,"confidence":"medium"}],"audio_stored_server_side":false,"cost":{"amount":0.05,"currency":"USD","breakdown":{"stt":0.04,"llm":0.01}}}`

## E. Constraints and budgets

- **Hard constraints:** No silent capture, ever: capture starts only on an explicit Record press in the focused app UI by the signed-in user; no API, schedule, calendar, remote, script or auto start; no hidden or background mode; the menu-bar recording indicator is visible for the whole capture and the operating system's own recording indicator is never suppressed. The consent reminder defaults on and cannot be disabled where policy requires all-party consent. On system sleep: finalise the current chunk, stop capture, mark `ended_by_sleep`, never auto-resume — a new Record press is required. A pending start that is older than 5 s, or that spans a sleep/wake, is discarded. Audio is relayed transiently to STT and never stored server-side; only the transcript is stored. The device token is used only for allowlisted actions. Never train on recorded audio; diarization labels are ephemeral and never become voiceprints; no sentiment or emotion scoring.
- **Soft constraints:** Prefer two-track for online meetings; chunk size about 5 minutes; keep post-processing short.
- **Resource and tenant scope:** the user's own device, spool and minutes only.
- **Data sources allowed:** the two capture tracks, the user's title and attendee list.
- **External transfers permitted:** audio chunks to the configured STT provider through the relay, transcript text to the configured LLM, solely to produce these minutes.
- **Time limit:** capture up to `max_duration_min` (`TBD`; the OS audio daemon holds its own power assertion, so an app cap bounds only the app); post-processing ≤ 15 minutes for a 1-hour meeting (`TBD` per provider).
- **Cost or token budget:** about USD 0.08 per meeting hour for STT in the reference; per-recording cap `TBD` (operator).
- **Maximum steps:** 11 (S1–S11).
- **Maximum retries and correction cycles:** chunk upload: 5 attempts with backoff while within spool retention; extraction: 1 retry; correction cycles: 1 per step, 3 per run.
- **Rate limits and concurrency:** one active recording per device; chunk uploads 2 in parallel.
- **Retention and logging constraints:** local spool deleted after successful ingest or after `spool_retention_days` (default 7), whichever comes first; transcripts and minutes per the operator's retention policy (`TBD`); logs never contain audio, transcript text or tokens.
- **Prohibited actions:** starting capture by any means other than a Record press; resuming after sleep without a new press; uploading audio for storage; using the device token for non-allowlisted actions; sharing minutes with third parties.
- **Stop conditions:** consent not confirmed; not paired or token revoked; permission missing; capture error; user cancels or discards.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Recording session and local chunks | Record press + consent (S2–S3) | Human action required (the press) | session file exists; indicator on |
| Create | Device pairing | user confirms the 6-digit code while signed in | Approval Required (code confirm) | pairing `confirmed` |
| Create | Transcript segments, minutes session | recording stopped; transcript merged | Allowed (user's own tenant) | `store.get` returns the session |
| Read | Recording status, own minutes, pairing status | own device / tenant | Allowed | — |
| Update | Recording session `ended_at`, `end_reason`; chunk `uploaded` | own session | Allowed | fields set once |
| Update | Pairing `revoked` (kill switch) | user or operator | Allowed | next relay call returns `401` |
| Delete | Local spool chunks (server-side minutes and transcripts: Not Allowed) | after ingest confirmed, after retention, or on user discard | Allowed (local, own data only) | files gone; session marked purged |
| External side effect | audio chunk to STT provider (transient) | paired, allowlisted, within budget | Allowed under the installation consent | relay receipt per chunk |
| External side effect | "minutes ready" message to the user | minutes stored | Allowed | delivery receipt |

- **Idempotency:** chunk key = hash(`device_id`, `recording_id`, `track`, `seq`); ingest key = hash(`device_id`, `recording_id`); one minutes session per `recording_id`.
- **Expected side effects:** local audio files during the retention window; STT and LLM charges; one minutes session.
- **Compensation:** a recording can be discarded before upload (local delete); revoking the device token stops further uploads at once.
- **Irreversible:** chunks already relayed to the STT provider; provider charges.
- **Ambiguous outcomes:** if an upload or ingest times out, query by idempotency key before resending; never create a second minutes session for the same `recording_id`.

## G. Autonomy and approval policy

- **Default autonomy level:** A1 — supervised execution.
- **Planning level:** A2 (readiness checks, chunk scheduling).
- **Execution level:** capture start is A0 — a human action only; transcription and minutes after Stop are A2 within the user's own tenant.
- **Verification level:** A2.
- **Recovery level:** A2 for bounded upload retries; A0 for anything that would capture again.
- **Approval required for:** pairing a device (code confirmation); disabling the consent reminder (only where policy allows); choosing mic-only when two-track is unavailable.
- **Actions never permitted:** any non-press start; auto-resume after sleep; suppressing either recording indicator; storing audio server-side; voiceprints; using the device token outside the allowlist.
- **Conditions requiring clarification:** unsupported language; mode conflicts with permissions.
- **Conditions requiring escalation:** capture produced an empty track; token revoked mid-upload; uploads still failing near spool expiry; a participant objects (the user stops and discards).
- **Safe state after stop, cancellation or escalation:** capture stopped, indicator off, chunks either uploaded or held in the local spool until retention, nothing restarted.

A Record press covers one recording. It does not cover a restart after sleep, a later meeting or any automatic start.

## H. Execution plan

**S1 — Check readiness**
1. Objective: confirm the device can record and upload before the user relies on it.
2. Capability: `nova.desktop.recorder.status`; relay health check (allowlisted, `binding://nova/desktop/stt-relay`).
3. Input mapping: `mode`, `language` → readiness query.
4. Dependencies: none (pairing done once beforehand via `nova.desktop.pairing.pair`).
5. Preconditions: the app is open and signed in.
6. Action / side effects: check pairing (token present, health returns 200), OS version, signed build, microphone and system-audio permission, free disk ≥ 500 MB (`TBD`); no side effects.
7. Autonomy: A2; no approval.
8. Timeout / retry: 10 s / 1 retry; `503 auth_unavailable` → show "server unreachable — recording will upload later", do not unpair.
9. Idempotency: read-only.
10. Postconditions: a readiness report with pass/fail per item.
11. Verification: every hard item passes; `401` is the only response treated as unpaired.
12. Failure routes: not paired or `401` → `blocked` with "pair this device"; system-audio permission missing in `two_track` → `blocked` with instructions or offer mic-only.
13. Escalation: none.
14. Evidence: readiness report (no token values).

**S2 — Consent reminder**
1. Objective: make sure everyone knows before capture begins.
2. Capability: app UI (local).
3. Input mapping: `consent_confirmed`.
4. Dependencies: S1.
5. Preconditions: readiness passed.
6. Action / side effects: show "I have told everyone this is recorded" with Confirm / Cancel; no side effects.
7. Autonomy: A0 — human decision.
8. Timeout / retry: none; the prompt simply stays until answered or closed.
9. Idempotency: one confirmation per recording.
10. Postconditions: `consent_confirmed = true` with timestamp, or cancelled.
11. Verification: confirmation timestamp precedes the press accepted in S3.
12. Failure routes: Cancel or close → `cancelled`, nothing captured.
13. Escalation: not applicable.
14. Evidence: consent confirmation time (no audio).

**S3 — Start on the Record press**
1. Objective: begin capture only because the user pressed Record.
2. Capability: `nova.desktop.recorder.start` (variant `two-track` or `mic-only`).
3. Input mapping: `record_press`, `mode`, `max_duration_min`.
4. Dependencies: S2.
5. Preconditions: consent confirmed; no active recording; press source is `ui_button` or `ui_shortcut` in the focused app; press age ≤ 5 s; no sleep/wake since the press.
6. Action / side effects: the sidecar opens the microphone and, in two-track, the system-audio tap; the menu-bar indicator turns on; the OS shows its own recording indicator; a local session file is created.
7. Autonomy: A0 trigger (human press); the start call itself runs under that press.
8. Timeout / retry: 5 s to confirm capture is live; no automatic retry — a failed start needs a new press.
9. Idempotency: one recording per press ID; a duplicate press while recording is ignored.
10. Postconditions: session `recording`, `indicator_shown = true`, track start offsets recorded.
11. Verification: status shows both indicators on and non-zero level on at least one track within 10 s.
12. Failure routes: stale or sleep-spanning press → discard and ask the user to press again; permission prompt still open → wait, then apply the stale rule; capture error → `failed`.
13. Escalation: none.
14. Evidence: press ID, accept time, indicator state.

**S4 — Capture and chunk**
1. Objective: write audio safely in small finalised pieces.
2. Capability: `nova.desktop.recorder.status` (monitoring); sidecar chunk writer.
3. Input mapping: active session.
4. Dependencies: S3.
5. Preconditions: session `recording`.
6. Action / side effects: finalise a chunk per track about every 5 minutes into the local spool; watch levels; keep the indicator on.
7. Autonomy: A2.
8. Timeout / retry: runs until S5; a chunk write failure retries once, then records a gap.
9. Idempotency: chunk `seq` is monotonic per track.
10. Postconditions: finalised chunks with SHA-256 and timing.
11. Verification: chunk timings are contiguous per track; a track below the silence threshold for its whole length is flagged `empty`.
12. Failure routes: disk full → stop capture (S5, `capture_error`) and keep what exists; permission revoked mid-call → S5 with `permission_revoked`.
13. Escalation: an empty track after 2 minutes → warn the user in the app while recording continues.
14. Evidence: chunk manifest (hashes, timings, levels).

**S5 — Stop**
1. Objective: end capture cleanly for every reason.
2. Capability: `nova.desktop.recorder.stop`.
3. Input mapping: `recording_id`, stop reason.
4. Dependencies: S4.
5. Preconditions: session `recording`.
6. Action / side effects: finalise the current chunk, release the taps, turn the indicator off. Reasons: Stop pressed (`user_stopped`), cap (`max_duration`), system sleep (`ended_by_sleep`), permission loss, app quit, capture error.
7. Autonomy: A2 (stopping is always allowed).
8. Timeout / retry: 3 s to finalise; on sleep the stop runs in the OS's will-sleep notification window.
9. Idempotency: stop on a stopped session is a no-op.
10. Postconditions: session `ended` with `end_reason`; on wake after `ended_by_sleep` the app shows "Recording ended when your computer slept" and does not restart.
11. Verification: no capture activity and no indicator after stop; the last chunk is finalised and hashed.
12. Failure routes: finalise fails → keep the partial chunk, record a gap.
13. Escalation: none.
14. Evidence: stop reason, final chunk hash, stop time.

**S6 — Transcribe chunks**
1. Objective: turn each chunk into timed text without storing audio server-side.
2. Capability: `nova.desktop.stt_relay.transcribe_chunk` → `nova.transcribe.audio.transcribe.whisper-hosted`.
3. Input mapping: chunk bytes, `track`, `seq`, `language`, chunk key.
4. Dependencies: S4 (may start during recording) and S5 (completes after stop).
5. Preconditions: device token valid; chunk finalised.
6. Action / side effects: chunk relayed to the STT provider and discarded by the relay after the response; charges accrue.
7. Autonomy: A2.
8. Timeout / retry: 60 s per chunk / 5 attempts with backoff; `503` → keep in spool and retry later; `401` → stop uploads.
9. Idempotency: chunk key; a replay returns the cached segments.
10. Postconditions: segments per chunk with times relative to the chunk start.
11. Verification: every non-empty chunk has a receipt; segment times fall within the chunk duration.
12. Failure routes: chunk still failing at spool expiry → record a gap, continue; `403 forbidden_for_device` → `blocked` (misconfiguration).
13. Escalation: `401` (revoked) mid-upload → tell the user; nothing more uploads.
14. Evidence: per-chunk receipt IDs (no audio, no text in logs).

**S7 — Merge tracks and attribute speakers**
1. Objective: one ordered transcript with speaker labels.
2. Capability: `nova.transcribe.audio.diarize.channel-split` (or `.none` in mic-only).
3. Input mapping: segments, track start offsets, chunk start times.
4. Dependencies: S6.
5. Preconditions: all chunks transcribed or marked as gaps.
6. Action / side effects: shift segment times by chunk and track offsets; label mic segments "Me", system segments "Others" (or "Room"); interleave by time; no side effects.
7. Autonomy: A2.
8. Timeout / retry: 30 s / 1 retry.
9. Idempotency: deterministic for the same inputs.
10. Postconditions: `transcript-segment@0.1.0` list ordered by `start_ms`.
11. Verification: no overlap errors beyond 500 ms caused by offset misalignment; gaps listed explicitly.
12. Failure routes: offsets missing → fall back to chunk start times and flag reduced alignment.
13. Escalation: none.
14. Evidence: merge report (segment counts per speaker, gaps).

**S8 — Submit the transcript**
1. Objective: hand the transcript to the minutes pipeline once.
2. Capability: `nova.desktop.minutes_ingest.submit`.
3. Input mapping: merged transcript, `title`, `attendees`, `mode`, `end_reason`, gaps, ingest key.
4. Dependencies: S7.
5. Preconditions: device token valid.
6. Action / side effects: creates one pending minutes session with transcript segments in the user's tenant.
7. Autonomy: A2.
8. Timeout / retry: 30 s / 3 attempts; on timeout query by ingest key first.
9. Idempotency: ingest key = hash(`device_id`, `recording_id`).
10. Postconditions: a `minutes_id`.
11. Verification: `nova.minutes.store.get` returns the session with the same segment count.
12. Failure routes: `401` → `blocked`, spool kept until retention; `503` → retry later.
13. Escalation: none.
14. Evidence: `minutes_id`, ingest receipt.

**S9 — Extract structured minutes**
1. Objective: summary points, decisions and action items.
2. Capability: `nova.minutes.extract.structured` (`single-pass` or `chunked` by transcript length).
3. Input mapping: transcript, `attendees`, meeting date.
4. Dependencies: S8.
5. Preconditions: transcript stored.
6. Action / side effects: LLM call; charges accrue.
7. Autonomy: A2.
8. Timeout / retry: 90 s / 1 retry.
9. Idempotency: keyed by `minutes_id`.
10. Postconditions: draft minutes; owners only "Me", "Others" or named attendees; due dates only when stated.
11. Verification: output parses against `minutes@0.1.0`.
12. Failure routes: unparseable twice → `partially_completed` with transcript only.
13. Escalation: none.
14. Evidence: model ID, prompt version, token counts.

**S10 — Verify grounding and store**
1. Objective: keep only what the transcript supports.
2. Capability: `nova.minutes.verify.grounding`; `nova.minutes.store.create`.
3. Input mapping: draft minutes, transcript.
4. Dependencies: S9.
5. Preconditions: draft minutes valid.
6. Action / side effects: each item gets `source_refs`; unsupported items are removed; weakly supported items get `confidence: low` and a flag; the minutes session is saved.
7. Autonomy: A2.
8. Timeout / retry: 60 s / 1 retry.
9. Idempotency: one minutes record per `minutes_id`.
10. Postconditions: stored minutes with 100 % of items carrying source references.
11. Verification: a separate verifier pass, not the extractor, checks every item (Section K).
12. Failure routes: verifier unavailable → store with every item marked `low` and disclose.
13. Escalation: none.
14. Evidence: item → segment references, removed-item count.

**S11 — Deliver and clean up**
1. Objective: tell the user and release local audio.
2. Capability: `nova.notify.owner.send`; local spool delete.
3. Input mapping: `recording_result`.
4. Dependencies: S10.
5. Preconditions: minutes stored, or the run ended earlier with a status.
6. Action / side effects: send "Minutes ready: Budget check-in (30 min, 2 action items)" with a link; delete spooled chunks whose content is in the stored transcript.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 2 retries.
9. Idempotency: message key = `recording_id`.
10. Postconditions: delivered; spool purged or scheduled for retention expiry.
11. Verification: delivery receipt; spool file count for this recording = 0 after a successful ingest.
12. Failure routes: delivery fails → result stays in the app and run record, `partially_completed`.
13. Escalation: none.
14. Evidence: message ID, purge record.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Start provenance and consent | capture began only after consent (when the reminder is on) and a `ui_button`/`ui_shortcut` press ≤ 5 s old, with no sleep between | consent and press logs vs session start | exact | discard start; incident logged |
| Indicator | menu-bar and OS indicators on for the whole capture | status samples | 100 % of samples | stop capture, report |
| Sleep | `ended_by_sleep` sessions never resume capture | session log | no capture after the sleep timestamp | incident, stop |
| Audio presence | no track silent for its entire length | chunk levels | none silent, or flagged `empty_tracks` | disclose; `failed` if all silent |
| Upload completeness | `chunks_transcribed` + gaps = `chunks_total` | relay receipts | equal | retry or record gaps |
| Device and tenant scope | token used only for allowlisted actions; minutes written only to the user's tenant | server log, store response | no `403`; tenant match | `blocked`, fix configuration, escalate |
| No audio stored | server holds transcript only | storage audit | `audio_stored_server_side = false` | escalate to operator |
| Grounding | every summary point, decision and action item has `source_refs` | verifier output | 100 % | remove or flag `low` |
| Output schema | `recording_result` valid | validator | valid | fix fields; never fake values |

A successful relay response proves only that a chunk was transcribed, not that the minutes are complete or correct.

## J. Self-correction and recovery

- **Failure classes:** validation, missing pairing, permission missing, stale press, capture error, empty track, timeout, transient relay error (`503`), revoked token (`401`), forbidden action (`403`), failed grounding, sleep interruption.
- **Permitted corrections:** retry chunk uploads and ingest by idempotency key within the spool window; re-run extraction once; fall back to chunk start times when track offsets are missing (flagged); process the captured portion after `ended_by_sleep`.
- **Forbidden corrections:** restart capture without a new press; auto-resume after sleep; switch two-track to mic-only silently; upload audio for storage; switch STT variant silently; keep ungrounded items as high confidence.
- **Maximum correction cycles:** 1 per step; 3 per run (upload retries counted separately, max 5 per chunk).
- **Reverification required after correction:** yes — re-run that step's verification, and the grounding check after any re-extraction.
- **Stop conditions:** correction budget exhausted; token revoked; consent withdrawn.
- **Recovery or compensation:** keep chunks in the local spool until retention; the user can discard a recording before upload.
- **What must be reported:** gaps, empty tracks, sleep interruptions, removed or low-confidence items, charges.

## K. Independent challenge

- **Assumptions to challenge:** that the system track holds only the other participants (notification sounds, media playback); that "Others" is one person; that every participant knew about the recording.
- **Alternative approach:** for a 1:1 phone-style call, a meeting app's own transcript (where the user's organisation provides it) may avoid capture entirely; mention it in the result when relevant.
- **Contradictory evidence:** decisions later reversed in the same transcript; action items with conflicting owners.
- **Could mislead:** minutes from a recording with a long gap or a silent track presented as complete; "Others" read as a single named person.
- **Separate verifier:** the grounding check runs as a separate pass from extraction; start provenance and indicator checks are rule-based on local logs, not LLM judgement.
- **Findings requiring disclosure:** gaps, `ended_by_sleep`, empty tracks, low-confidence items, mic-only mode.

## L. Human escalation

- **Triggers:** empty track detected; token revoked mid-upload; uploads failing as spool expiry nears; a participant objects; storage audit finds audio on the server.
- **Why the agent cannot decide:** consent, legal exposure and data loss belong to the user or operator.
- **Information presented:** recording title, duration, end reason, chunk status, gaps, spool expiry time.
- **Decision requested:** discard the recording; re-pair and upload; accept partial minutes; keep the spool longer within policy.
- **Options and consequences:** discarding deletes local audio permanently; accepting partial minutes leaves gaps disclosed.
- **Safe behaviour while waiting:** no capture; chunks held locally; nothing re-uploaded after a `401`.
- **If no response:** at spool expiry, chunks are deleted and the run ends `partially_completed` or `failed` with the evidence saved.
- **Resume:** after re-pairing, S1 runs again; uploads resume by idempotency key; capture never resumes.

## M. Completion conditions

Mandatory: `recording_result` valid; capture started only from a valid press and stayed behind visible indicators; no server-side audio; every stored item grounded; gaps and limitations disclosed; evidence references present.

- `completed` — recorded on a press, stopped by the user or cap, every chunk transcribed, minutes grounded and stored, user notified.
- `partially_completed` — `ended_by_sleep`, gaps, an empty track, mic-only limits, extraction fallback to transcript only, or delivery failed. Such a run is never reported as `completed`.
- `blocked` — not paired, token revoked, permission or signed build missing, consent not confirmed, or a non-press start attempt was rejected.
- `failed` — capture error before any usable audio, all tracks silent, or chunks expired untranscribed.
- `cancelled` — the user cancelled at the consent reminder or discarded the recording before upload.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome; `recording_result`; step statuses S1–S11; verification results and failed checks; evidence references (press ID, consent time, chunk manifest, relay receipts, `minutes_id`, grounding references); approvals and scope (pairing, mode choice); side effects (STT and LLM charges, local spool state) and reconciliation status; corrections performed; unresolved challenges; limitations (gaps, "Others" label); escalation state; learning candidate. Never include audio, device tokens or full transcripts in shared channels.

## O. Learning and revision

- **Outcome metrics:** recordings per week, sleep-interruption rate, empty-track rate, chunk retry rate, grounding removal rate, STT cost per hour, time from Stop to minutes.
- **Evidence retained:** run record, chunk manifest (hashes and timings only), grounding report, per retention policy.
- **Data minimisation:** metrics stored without transcript text; local audio deleted after ingest or retention.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}`.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests, including T1–T6.
- **Promotion approval:** publisher review plus Skill Port review; no change may add a non-press start or weaken the indicators.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (two test accounts owned by the tester, fictional script) | two-track recording, all chunks transcribed, grounded minutes stored, user notified | UNVERIFIED — capture not yet tested on the target computer; server ingest observed in the reference |
| Invalid and missing input (unsupported language; title over 120 chars; consent not confirmed) | stops before capture / truncates with notice / no capture and reminder shown again | TBD |
| Non-press start (API, schedule, remote, script) | rejected, logged, nothing captured | TBD |
| Missing capability (relay binding absent) | recording stays local, `blocked` for upload with a dependency message | TBD |
| Permission denied (system-audio permission missing or unsigned build) | `blocked` with instructions; no silent mic-only | TBD |
| Unauthorized scope (device token on a non-allowlisted action) | `403 forbidden_for_device` | Observed in the reference; TBD on Skill Port |
| Revoked token | `401`; uploads stop; spool kept until retention | TBD |
| Auth backend outage | `503 auth_unavailable`; device stays paired; retries later | Observed in the reference; TBD on Skill Port |
| Tenant isolation | device cannot read or write another tenant's minutes | TBD |
| Timeout / retry on chunk upload | re-query by key, then resend; no duplicate segments | TBD |
| Duplicate / replay (chunk and ingest) | same segments; same `minutes_id` | TBD |
| T1 — sleep mid-recording | chunk finalised, capture stops, `ended_by_sleep`, indicator off | TBD (pending on the target computer) |
| T2 — wake after sleep | no auto-resume; notice shown; new press required | TBD |
| T3 — upload after wake | captured part transcribed; gap disclosed | TBD |
| T4 — press during a microphone-permission prompt, then sleep | pending start discarded on wake | TBD |
| T5 — offline at wake | spool retried; no loss within retention | TBD |
| T6 — sleep while idle | nothing starts on wake | TBD |
| False success (relay OK, track silent) | `empty_tracks` flagged; not `completed` | TBD |
| Failed verification (ungrounded action item) | item removed or flagged `low` | TBD |
| Correction budget exhausted | stops safely; partial result reported | TBD |
| Escalation (token revoked mid-upload) | user told; nothing more uploads | TBD |
| Cancellation (cancel at consent; discard before upload) | `cancelled`; local files deleted; nothing sent | TBD |
| Rollback / compensation (discard after partial upload) | remaining spool deleted; relayed chunks reported | TBD |
| Output schema and status | valid `recording_result`, correct status | TBD |
| Compatibility / regression (new OS version, new STT model) | capture and offsets still correct | TBD |

All tests use the tester's own test accounts and fictional scripts; never record real meetings or people who have not agreed. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, schema registry and local-IPC representation — TBD (owner: publisher, with Skill Port).
- Maximum recording length is an open product decision; an app cap bounds only the app because the OS audio daemon holds its own power assertion (owner: product owner).
- Sleep-safe stop is written and reviewed but not applied until tests T1–T6 pass on the target computer (owner: publisher).
- Signed-build requirement: the system-audio permission prompt needs a signed build; distribution signing is a separate cost (owner: publisher).
- "Others" groups every remote speaker (per-person attribution in group calls needs a diarization model not available here); desktop operating systems with an app-audio capture API only — no mobile path exists.
- Retention period for transcripts and minutes — TBD (owner: operator, per local law). Recording consent rules differ by jurisdiction; some require every participant's consent. The user remains responsible; the runtime keeps the reminder on where policy requires it.
