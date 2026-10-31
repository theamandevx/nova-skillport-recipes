# Prepare a call

## A. Identity and ownership

- **Recipe ID:** `nova.comms.prepare-a-call` (proposed until assigned by Skill Port)
- **Name:** Prepare a call
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** productivity
- **Tags:** voice, outbound-call, telephony, transcript, call-summary, human-approval · Vedaxi task type: Prepare a call
- **Summary:** On an explicit request, the agent resolves who to call, prepares a short call brief, gets the user's approval, places one outbound AI voice call that pursues the stated goal, and returns the outcome with transcript, recording reference and cost.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** A telephony provider with bidirectional real-time audio streaming and a rented caller ID; a speech-to-text provider; a text-to-speech provider; a fast tool-calling LLM; a contact store; an owner notification channel. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — the end-to-end flow runs in Nova's reference implementation (placing calls, live conversation, transcript, recording, cost and recap). Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.contacts.directory` | Capability | Find a person's phone number from approved sources | medium | UNVERIFIED |
| `nova.contacts.directory.lookup` | Operation (read) | Look a name up in assistant memory, then the contact directory | medium | UNVERIFIED |
| `nova.contacts.directory.save` | Operation (create) | Save a newly provided number, with user consent | low | UNVERIFIED |
| `nova.voice.config` | Capability | List voices and provider readiness | low | UNVERIFIED |
| `nova.voice.config.list_voices` | Operation (read) | Return selectable voices with `ready` and `reason` per provider | low | UNVERIFIED |
| `nova.voice.call` | Capability | Place, monitor and end one outbound AI voice call | high | UNVERIFIED |
| `nova.voice.call.place` | Operation (create + external side effect) | Dial the recipient and start the conversation pipeline | high | UNVERIFIED |
| `nova.voice.call.status` | Operation (read) | Return the call state (`queued`, `ringing`, `in_progress`, `ended`) | low | UNVERIFIED |
| `nova.voice.call.hangup` | Operation (update + external side effect) | End a live call immediately | medium | UNVERIFIED |
| `nova.voice.records` | Capability | Read the call record, transcript, recording reference and cost | medium | UNVERIFIED |
| `nova.voice.records.get` | Operation (read) | Return one call's record with transcript lines and cost breakdown | medium | UNVERIFIED |
| `nova.voice.recap` | Capability | Summarise a finished call into outcome, bullets and follow-ups | low | UNVERIFIED |
| `nova.voice.recap.summarise` | Operation (read, LLM) | Produce a structured recap from the transcript | low | UNVERIFIED |
| `nova.notify.owner` | Capability | Deliver a message to the requesting user's own channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Send the result to the user's chat | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.contacts.directory.lookup.memory` — assistant memory first; read-only. Selected first.
- `nova.contacts.directory.lookup.directory` — contact directory second; read-only. Eligible only if memory has no match.
- `nova.contacts.directory.lookup.ask-user` — human-assisted: ask the user once for the number. Eligible only if both read-only variants miss. Never guess a number.
- `nova.voice.call.place.default` — telephony provider with bidirectional audio streaming → speech-to-text → tool-calling LLM → text-to-speech, with voice-activity detection, barge-in, an idle hang-up and a maximum-duration cap. Binding `binding://nova/voice/outbound-call` (UNVERIFIED), protocol: internal HTTP API exposed by the calling engine; mode: async.
- `nova.voice.config.list_voices.default` — one variant per TTS provider is selectable only when that provider reports `ready: true`. A not-ready provider is rejected with its reason; there is no silent fallback to a different provider, because a voice change is user-visible.

**Bindings:** `binding://nova/contacts/lookup`, `binding://nova/contacts/save`, `binding://nova/voice/voices`, `binding://nova/voice/outbound-call`, `binding://nova/voice/call-status`, `binding://nova/voice/hangup`, `binding://nova/voice/records`, `binding://nova/voice/recap`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: a self-hosted calling engine behind an authenticated HTTPS endpoint, with call records in a database exposed only through security-definer functions.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Contact | `schema://nova/prepare-a-call/contact@0.1.0` | the requesting user | read; create (consent) |
| Call | `schema://nova/prepare-a-call/call-record@0.1.0` | the requesting user's tenant | create, read, update (end only) |
| Transcript line | `schema://nova/prepare-a-call/transcript-line@0.1.0` | linked to one Call | read |
| Recording | reference only (URL or storage key) | linked to one Call | read |
| Recap | `schema://nova/prepare-a-call/recap@0.1.0` | linked to one Call | create |

**Authorization requirements (declared, never a grant):** `contacts:read`, `contacts:write` (save only, consent required), `voice:call:create`, `voice:call:read`, `voice:call:end`, `notify:owner`. Telephony account credentials live in the runtime as `secret_ref://telephony-account`; speech and LLM keys as `secret_ref://stt`, `secret_ref://tts`, `secret_ref://llm`.

**Policy / trust:** risk class `high` (real-time contact with a third party). Requires the per-call approval in step S5, a country-code allowlist, quiet hours and a per-call cost cap.

## C. Intent and applicability

- **User problem:** The user needs a short phone conversation done (confirm a time, pass on a message, ask a simple question) without making the call personally.
- **Desired outcome:** One call placed to the right person, the goal pursued politely, and a truthful result with evidence.
- **Trigger or entry conditions:** The user explicitly asks the agent to call a named person or number and states what to say or ask.
- **Applicable situations:** Appointment confirmations, reminders, simple information requests, relaying a message.
- **Out-of-scope situations:** Inbound calls; bulk, marketing or cold calls; emergency services; collecting payment or sensitive data; negotiating contracts; impersonating anyone; calling without an explicit request; calling numbers outside the allowlist.
- **Required inputs:** contact, goal (see Section D).
- **Prerequisites:** A verified telephony account with a caller ID permitted for the destination country; ready STT, TTS and LLM providers; recording disclosure configured for the jurisdiction.
- **Required capabilities:** `nova.contacts.directory`, `nova.voice.config`, `nova.voice.call`, `nova.voice.records`, `nova.voice.recap`, `nova.notify.owner` (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port; every binding is a dependency to resolve before activation.
- **Assumptions:** Verified in the reference implementation: outbound calling, live transcript, recording, cost and recap. Assumed: the target runtime offers equivalent providers in the user's region.
- **Ambiguity policy:** Ask. If the person, number or goal is unclear, ask one clarifying question; never guess a number or invent a goal.
- **Vedaxi task type:** Prepare a call.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `contact` | string: a name, or an E.164 number (`^\+[1-9][0-9]{7,14}$`) | yes | user; personal data | — | Ask once. Invalid number format: reject and ask. Name with two directory matches: ask which one. |
| `goal` | string, 10–500 chars | yes | user; may be personal | — | Ask for it; never call without a goal. |
| `language` | enum `en`, `hi`, `hi-en`, other runtime-supported codes | no | user | runtime default | Unsupported: stop and say which languages are available. |
| `voice_id` | string from `list_voices` | no | user | runtime default voice | Not ready or unknown: reject with the provider's reason; do not switch silently. |
| `max_duration_s` | integer 60–600 | no | user | 300 | Out of range: clamp and tell the user. |
| `recording_disclosure` | boolean | no | runtime policy | `true` | Can be `false` only where policy allows; otherwise the runtime value wins. |

```json
{"$id":"schema://nova/prepare-a-call/request@0.1.0","type":"object","required":["contact","goal"],
 "properties":{"contact":{"type":"string","minLength":2},"goal":{"type":"string","minLength":10,"maxLength":500},
 "language":{"type":"string"},"voice_id":{"type":"string"},"max_duration_s":{"type":"integer","minimum":60,"maximum":600},
 "recording_disclosure":{"type":"boolean"}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `call_result` | object `schema://nova/prepare-a-call/call-result@0.1.0` | `call_id`, `status`, `connected`, `end_reason`, `outcome` | the requesting user's channel and the run record | personal data; owner only | cross-checked against the call record and transcript (Section I) |

```json
{"$id":"schema://nova/prepare-a-call/call-result@0.1.0","type":"object",
 "required":["call_id","status","connected","end_reason","outcome"],
 "properties":{"call_id":{"type":"string"},"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},
 "connected":{"type":"boolean"},"duration_s":{"type":"integer","minimum":0},
 "end_reason":{"enum":["completed","no_answer","busy","caller_hung_up","agent_ended","idle_timeout","max_duration","cut_by_user","error"]},
 "outcome":{"enum":["goal_met","goal_partly_met","goal_not_met","not_connected"]},
 "summary":{"type":"array","items":{"type":"string"},"maxItems":6},"follow_ups":{"type":"array","items":{"type":"string"}},
 "transcript_ref":{"type":"string"},"recording_ref":{"type":["string","null"]},
 "cost":{"type":"object","properties":{"amount":{"type":"number"},"currency":{"type":"string"},"breakdown":{"type":"object"}}}}}
```

Other schemas used by the capability register (Section B and `manifest.draft.yaml`):

```json
{"$id":"schema://nova/prepare-a-call/contact-query@0.1.0","type":"object","required":["query"],
 "properties":{"query":{"type":"string","minLength":2},"sources":{"type":"array","items":{"enum":["memory","directory","user"]}}}}
{"$id":"schema://nova/prepare-a-call/contact@0.1.0","type":"object","required":["name","phone_e164","source"],
 "properties":{"name":{"type":"string"},"phone_e164":{"type":"string","pattern":"^\\+[1-9][0-9]{7,14}$"},
 "source":{"enum":["memory","directory","user"]},"matches":{"type":"integer","minimum":0}}}
{"$id":"schema://nova/prepare-a-call/voice-query@0.1.0","type":"object",
 "properties":{"language":{"type":"string"},"provider":{"type":"string"}}}
{"$id":"schema://nova/prepare-a-call/voice-list@0.1.0","type":"object","required":["voices"],
 "properties":{"voices":{"type":"array","items":{"type":"object","required":["voice_id","provider","ready"],
 "properties":{"voice_id":{"type":"string"},"label":{"type":"string"},"provider":{"type":"string"},"languages":{"type":"array","items":{"type":"string"}},
 "ready":{"type":"boolean"},"not_ready_reason":{"type":["string","null"]}}}}}}
{"$id":"schema://nova/prepare-a-call/place-input@0.1.0","type":"object",
 "required":["phone_e164","brief","language","voice_id","max_duration_s","idempotency_key","approval_id"],
 "properties":{"phone_e164":{"type":"string","pattern":"^\\+[1-9][0-9]{7,14}$"},"brief":{"type":"string","maxLength":2000},
 "language":{"type":"string"},"voice_id":{"type":"string"},"max_duration_s":{"type":"integer","minimum":60,"maximum":600},
 "recording_disclosure":{"type":"boolean"},"idempotency_key":{"type":"string"},"approval_id":{"type":"string"}}}
{"$id":"schema://nova/prepare-a-call/call-ref@0.1.0","type":"object","required":["call_id"],
 "properties":{"call_id":{"type":"string"},"idempotency_key":{"type":"string"}}}
{"$id":"schema://nova/prepare-a-call/call-record@0.1.0","type":"object","required":["call_id","to_e164","status"],
 "properties":{"call_id":{"type":"string"},"provider_ref":{"type":["string","null"]},"to_e164":{"type":"string"},
 "status":{"enum":["queued","ringing","in_progress","ended"]},"started_at":{"type":"string","format":"date-time"},
 "answered_at":{"type":["string","null"],"format":"date-time"},"ended_at":{"type":["string","null"],"format":"date-time"},
 "end_reason":{"enum":["completed","no_answer","busy","caller_hung_up","agent_ended","idle_timeout","max_duration","cut_by_user","error",null]},
 "transcript":{"type":"array","items":{"$ref":"schema://nova/prepare-a-call/transcript-line@0.1.0"}},
 "recording_ref":{"type":["string","null"]},
 "cost":{"type":"object","properties":{"amount":{"type":"number"},"currency":{"type":"string"},"breakdown":{"type":"object"}}}}}
{"$id":"schema://nova/prepare-a-call/transcript-line@0.1.0","type":"object","required":["call_id","seq","role","text"],
 "properties":{"call_id":{"type":"string"},"seq":{"type":"integer","minimum":0},"role":{"enum":["agent","callee"]},
 "text":{"type":"string"},"at":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/prepare-a-call/recap@0.1.0","type":"object","required":["outcome","summary","follow_ups"],
 "properties":{"outcome":{"enum":["goal_met","goal_partly_met","goal_not_met","not_connected"]},
 "summary":{"type":"array","items":{"type":"string"},"maxItems":6},"follow_ups":{"type":"array","items":{"type":"string"}},
 "unsupported_claims":{"type":"array","items":{"type":"string"}}}}
{"$id":"schema://nova/prepare-a-call/delivery@0.1.0","type":"object","required":["delivered","channel"],
 "properties":{"delivered":{"type":"boolean"},"channel":{"type":"string"},"message_ref":{"type":["string","null"]},
 "error":{"type":["string","null"]}}}
```

**Example (fictional)**

Input: `{"contact":"Jane Doe","goal":"Confirm she can make Thursday 4 pm and ask if she needs parking.","language":"en"}`

Output: `{"call_id":"call_0001","status":"completed","connected":true,"duration_s":74,"end_reason":"agent_ended","outcome":"goal_met","summary":["Jane confirmed Thursday 4 pm.","She will drive and needs parking."],"follow_ups":["Reserve a parking space for Thursday."],"transcript_ref":"run://call_0001/transcript","recording_ref":"run://call_0001/recording","cost":{"amount":0.21,"currency":"USD","breakdown":{"telephony":0.09,"stt":0.04,"tts":0.05,"llm":0.03}}}`

## E. Constraints and budgets

- **Hard constraints:** explicit user request; per-call approval at S5; destination country code on the allowlist; never call the requesting user's own number unless `self_test` is set by the user; one active call per recipient; honour quiet hours (default 09:00–20:00 in the recipient's time zone); recording disclosure follows policy (default on); never impersonate a human or a third party — the agent identifies itself as an AI assistant calling on the user's behalf; never collect payment details, passwords or one-time codes.
- **Soft constraints:** keep calls short; use the user's preferred language and voice; end politely once the goal is met.
- **Resource and tenant scope:** the requesting user's contacts and call records only.
- **Data sources allowed:** assistant memory, the user's contact directory, the user's own message.
- **External transfers permitted:** audio to the configured telephony, STT, TTS and LLM providers, solely to run this call.
- **Time limit:** `max_duration_s` (default 300, ceiling 600); whole run ≤ 15 minutes.
- **Cost or token budget:** per-call cap `TBD` (set by the operator; the reference implementation shows an estimate before approval). Stop if the estimate exceeds the cap.
- **Maximum steps:** 11 (S1–S11).
- **Maximum retries and correction cycles:** dialling: 0 automatic redials; other read steps: 2 retries; correction cycles: 1.
- **Rate limits and concurrency:** 1 concurrent call per user unless the operator raises it; provider rate limits apply.
- **Retention and logging constraints:** transcripts and recordings kept per the operator's retention policy (`TBD`); logs never contain credentials or full recordings.
- **Prohibited actions:** redialling without new approval; changing the goal mid-call beyond the approved brief; sharing the user's private information that is not in the brief.
- **Stop conditions:** approval declined; policy check fails; provider not ready; cost estimate over cap; callee asks not to be called (record the opt-out).

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Call (one per approval) | S1–S5 passed | Approval Required (S5) | call record exists with the provider's call ID |
| Create | Contact | user supplied a new number and agreed to save it | Approval Required | record readable via `lookup` |
| Create | Recap | call ended | Allowed | recap fields match the transcript |
| Read | Contact, Call, Transcript, Recording reference, voice list | own tenant only | Allowed | — |
| Update | Call: `status`, `ended_at`, `end_reason` (by hang-up only) | call live | Allowed (to end a call) | status `ended` |
| Delete | any | — | Not Allowed | — |
| External side effect | outbound phone call | approved brief, allowlist, quiet hours, cap | Approval Required | provider confirms `in_progress` then `ended` |
| External side effect | result message to the requesting user | call finished | Allowed | delivery receipt |

- **Idempotency:** idempotency key = hash(`run_id`, `to_e164`, `approval_id`). A second `place` with the same key returns the existing call instead of dialling again.
- **Expected side effects:** the recipient's phone rings; provider charges accrue; a recording is created if recording is enabled.
- **Compensation:** a live call can be ended with `hangup`; charges and an answered call cannot be undone.
- **Irreversible:** the call itself, anything said on it, provider charges.
- **Ambiguous outcomes:** if `place` times out, read `status` by idempotency key before any further action; never redial on an unknown outcome.

## G. Autonomy and approval policy

- **Default autonomy level:** A1 — supervised execution.
- **Planning level:** A2 (resolve contact, run policy checks, draft the brief).
- **Execution level:** A1 — dialling only after the S5 approval; the live conversation runs within the approved brief.
- **Verification level:** A2.
- **Recovery level:** A1 — no redial without a new approval.
- **Approval required for:** placing the call (exact number, goal, language, voice, duration cap, cost estimate); saving a new contact.
- **Actions never permitted:** calling without approval; bulk or marketing calls; calling emergency numbers; impersonation; collecting payment data, passwords or one-time codes; disabling a recording disclosure that policy requires.
- **Conditions requiring clarification:** unknown or ambiguous contact; missing goal; unsupported language.
- **Conditions requiring escalation:** callee in distress or asks for a human; legal or medical topics arise; provider errors persist; cost cap reached.
- **Safe state after stop, cancellation or escalation:** no call in progress (hang up if live), nothing redialled, the user told what happened.

Approval covers one call with the stated brief. It does not cover redials, other numbers or later calls.

## H. Execution plan

**S1 — Validate the request**
1. Objective: confirm contact and goal are present and valid.
2. Capability: none (local validation).
3. Input mapping: user message → `request@0.1.0`.
4. Dependencies: none.
5. Preconditions: explicit request to call.
6. Action / side effects: parse and validate; no side effects.
7. Autonomy: A2; no approval.
8. Timeout / retry: 5 s / none.
9. Idempotency: not applicable.
10. Postconditions: a schema-valid request.
11. Verification: JSON Schema validation passes.
12. Failure routes: missing or invalid field → ask once (Ambiguity policy) → if still missing, `blocked`.
13. Escalation triggers: none.
14. Evidence: validated request object.

**S2 — Resolve the contact**
1. Objective: turn a name into exactly one E.164 number.
2. Capability: `nova.contacts.directory.lookup` (variants memory → directory → ask-user).
3. Input mapping: `contact` → `query`.
4. Dependencies: S1.
5. Preconditions: `contact` is a name (skip if already a number).
6. Action / side effects: read-only lookups; asking the user is a message to the user only.
7. Autonomy: A2 for reads; A0 for ask-user.
8. Timeout / retry: 10 s per variant / 2 retries on transient errors.
9. Idempotency: read-only.
10. Postconditions: one `contact@0.1.0` with `phone_e164`.
11. Verification: number matches the E.164 pattern; exactly one match.
12. Failure routes: multiple matches → ask which; none → ask-user; user declines → `blocked`.
13. Escalation: none.
14. Evidence: which source supplied the number (not the full directory).

**S3 — Policy checks**
1. Objective: confirm the call is allowed now.
2. Capability: runtime policy (UNVERIFIED binding); `nova.voice.config.list_voices`.
3. Input mapping: `phone_e164`, `language`, `voice_id`, recipient time zone (derived from the country code; ask if ambiguous).
4. Dependencies: S2.
5. Preconditions: number resolved.
6. Action: check the allowlist, quiet hours, self-number guard, one-active-call rule, provider readiness, and the cost estimate against the cap.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 1 retry for provider readiness.
9. Idempotency: read-only.
10. Postconditions: every check passes, or a named failure.
11. Verification: each check records pass/fail with its reason.
12. Failure routes: any hard check fails → `blocked` with the reason (for example "outside quiet hours — next window 09:00 recipient time").
13. Escalation: cost estimate over cap → ask the user.
14. Evidence: the check results.

**S4 — Draft the call brief**
1. Objective: a short brief the voice agent follows.
2. Capability: LLM drafting (part of the planner).
3. Input mapping: `goal`, `language`, contact first name, the user's display name.
4. Dependencies: S3.
5. Preconditions: checks passed.
6. Action: write an opening (AI-assistant identification and, when enabled, the recording disclosure), the questions to ask, what to do if the person is unavailable, and when to end. In languages that use a non-Latin script, text-to-speech text is written in that script.
7. Autonomy: A2.
8. Timeout / retry: 30 s / 1 retry.
9. Idempotency: not applicable.
10. Postconditions: brief ≤ 1,200 characters, containing only facts from the goal.
11. Verification: the brief adds no facts or promises beyond the goal; disclosure present when required.
12. Failure routes: verification fails → redraft once → else `blocked`.
13. Escalation: none.
14. Evidence: the brief text.

**S5 — Approval checkpoint**
1. Objective: get explicit approval for this exact call.
2. Capability: user interaction.
3. Input mapping: number (masked except the last 4 digits in shared channels), contact name, brief, language, voice, duration cap, cost estimate.
4. Dependencies: S4.
5. Preconditions: brief verified.
6. Action: ask "Call Jane Doe (…0142) now with this brief?" with Approve / Edit / Cancel.
7. Autonomy: A0 — human decision.
8. Timeout / retry: wait up to 30 minutes, then `cancelled`.
9. Idempotency: the approval ID binds one call.
10. Postconditions: an approval record with ID, scope and timestamp.
11. Verification: the approval references this brief's hash.
12. Failure routes: Edit → back to S4; Cancel or timeout → `cancelled`.
13. Escalation: not applicable.
14. Evidence: approval ID, timestamp, brief hash.

**S6 — Place the call**
1. Objective: start exactly one outbound call.
2. Capability: `nova.voice.call.place.default`.
3. Input mapping: `phone_e164`, brief, language, voice, `max_duration_s`, idempotency key.
4. Dependencies: S5.
5. Preconditions: approval valid and unexpired; the S3 checks still pass (re-run quiet hours and the one-active-call rule).
6. Action / side effects: the recipient's phone rings; charges start once connected.
7. Autonomy: A1, covered by the S5 approval.
8. Timeout / retry: 20 s for the API call; **no automatic redial**.
9. Idempotency: key = hash(`run_id`, `to_e164`, `approval_id`).
10. Postconditions: a call record with the provider call ID in `queued` or `ringing`.
11. Verification: the record exists and its `to` equals the approved number.
12. Failure routes: API timeout → S7 status read by key; provider rejection → `failed` with the provider reason.
13. Escalation: repeated provider errors → tell the user.
14. Evidence: call ID, provider reference, timestamps.

**S7 — Monitor the call**
1. Objective: track the call truthfully until it ends.
2. Capability: `nova.voice.call.status`.
3. Input mapping: `call_id`.
4. Dependencies: S6.
5. Preconditions: a call record exists.
6. Action: poll or subscribe; `queued` is never reported as connected.
7. Autonomy: A2.
8. Timeout / retry: until `ended` or `max_duration_s + 60 s`.
9. Idempotency: read-only.
10. Postconditions: final state `ended` with an `end_reason`.
11. Verification: `answered_at` is set if and only if `connected = true`.
12. Failure routes: no final state by the deadline → `hangup`, then read again.
13. Escalation: the user asks to cut → `hangup` immediately.
14. Evidence: state transitions with timestamps.

**S8 — Live conversation (runs inside the calling engine)**
1. Objective: pursue the goal within the brief.
2. Capability: the `nova.voice.call.place.default` pipeline (STT → LLM → TTS, barge-in).
3. Input mapping: brief.
4. Dependencies: S6.
5. Preconditions: the call is answered.
6. Action: speak the opening; ask the questions; listen; end with the `end_call` tool once the goal is met or the callee wants to stop. An idle callee gets one "are you still there?" and then the call ends.
7. Autonomy: A1, bounded by the brief.
8. Timeout / retry: idle timeout ≈ 20 s; hard cap `max_duration_s`.
9. Idempotency: not applicable.
10. Postconditions: the call ended by the agent, the callee or a cap.
11. Verification: the transcript shows the disclosure (when required) and no out-of-brief commitments.
12. Failure routes: the callee asks for a human or is distressed → end politely and escalate.
13. Escalation: as above.
14. Evidence: transcript lines.

**S9 — Collect records**
1. Objective: gather the transcript, recording reference and cost.
2. Capability: `nova.voice.records.get`.
3. Input mapping: `call_id`.
4. Dependencies: S7.
5. Preconditions: call `ended`.
6. Action: read-only.
7. Autonomy: A2.
8. Timeout / retry: 30 s / 2 retries (the recording can arrive late).
9. Idempotency: read-only.
10. Postconditions: transcript (possibly empty if not connected), recording reference or null, cost.
11. Verification: transcript line count > 0 when `connected = true`.
12. Failure routes: recording missing after retries → set `recording_ref = null` and disclose.
13. Escalation: none.
14. Evidence: record references.

**S10 — Recap and verify**
1. Objective: an honest structured outcome.
2. Capability: `nova.voice.recap.summarise`.
3. Input mapping: transcript, goal.
4. Dependencies: S9.
5. Preconditions: records collected.
6. Action: produce outcome, at most 6 bullets and follow-ups.
7. Autonomy: A2.
8. Timeout / retry: 30 s / 1 retry.
9. Idempotency: not applicable.
10. Postconditions: `recap@0.1.0`.
11. Verification: every bullet is supported by a transcript line; `goal_met` only if the transcript shows it (Section K).
12. Failure routes: unsupported claim → regenerate once → else lower to `goal_partly_met` and disclose.
13. Escalation: none.
14. Evidence: bullet → transcript line references.

**S11 — Deliver the result**
1. Objective: tell the user what happened.
2. Capability: `nova.notify.owner.send`; optionally `nova.contacts.directory.save`.
3. Input mapping: `call_result`.
4. Dependencies: S10.
5. Preconditions: result assembled.
6. Action: send the result to the requesting user; offer to save a newly provided number.
7. Autonomy: A1; saving needs a yes.
8. Timeout / retry: 15 s / 2 retries.
9. Idempotency: message key = `run_id`.
10. Postconditions: delivered; contact saved only if approved.
11. Verification: delivery receipt.
12. Failure routes: delivery fails → keep the result in the run record and mark the run `partially_completed`.
13. Escalation: none.
14. Evidence: message ID.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Input validity | request matches `request@0.1.0` | validator | all required fields valid | ask once, then `blocked` |
| Number identity | dialled number = approved number | call record vs approval | exact match | hang up, `failed`, report |
| Policy | allowlist, quiet hours, cap, self-guard pass at S3 and again at S6 | policy log | all pass | `blocked` |
| Approval scope | approval hash = brief hash | approval record | equal | stop before dialling |
| Connection truth | `connected = true` only with `answered_at` set | provider status | consistent | correct the result |
| Disclosure | disclosure line in the first agent turn when required | transcript | present | flag a policy breach and escalate |
| Recap support | each bullet maps to transcript lines | recap references | 100 % mapped | regenerate or downgrade |
| Output schema | `call_result` valid | validator | valid | fix fields; never fake values |
| Data boundary | no data beyond the brief disclosed | transcript review | none found | escalate |

A provider returning success for `place` proves only that dialling started, not that the goal was met.

## J. Self-correction and recovery

- **Failure classes:** validation, missing contact, policy block, provider not ready, timeout, transient provider error, permission denial, failed assertion, ambiguous call state.
- **Permitted corrections:** re-ask for missing input once; retry read operations (bounded); redraft the brief once; reconcile an ambiguous `place` by reading status.
- **Forbidden corrections:** redial without new approval; switch to an unapproved voice, provider or number; drop the recording disclosure; inflate the outcome.
- **Maximum correction cycles:** 1 per step; 3 per run.
- **Reverification required after correction:** yes — re-run that step's verification and S3 policy checks before S6.
- **Stop conditions:** correction budget exhausted; hard constraint fails.
- **Recovery or compensation:** hang up a live call; report charges incurred.
- **What must be reported:** every correction, any call placed, charges, and any unresolved state.

## K. Independent challenge

- **Assumptions to challenge:** the number belongs to the intended person; the callee understood the questions; "yes" answered the actual question.
- **Alternative approach:** would a message (SMS or WhatsApp draft) serve the goal with less intrusion? Offer it when the call failed to connect.
- **Contradictory evidence:** transcript turns where the callee disagreed or was unsure.
- **Could mislead:** a recap reading `goal_met` when the transcript shows hesitation or a wrong person.
- **Separate verifier:** a second LLM pass (or rule check) compares the recap with the transcript independently of the summariser.
- **Findings requiring disclosure:** wrong person reached, partial answers, missing recording.

## L. Human escalation

- **Triggers:** ambiguous contact; cost estimate over cap; callee distressed or asks for a human; legal or medical topic; repeated provider failure; wrong-number suspicion.
- **Why the agent cannot decide:** identity, money, wellbeing or legal exposure.
- **Information presented:** contact, brief, policy results, call state, transcript excerpt, cost so far.
- **Decision requested:** approve / edit / cancel; call back personally; accept the partial result.
- **Options and consequences:** stated per trigger (for example, approving over the cap spends more than the default).
- **Safe behaviour while waiting:** no call in progress; nothing redialled.
- **If no response:** after 30 minutes the run ends `cancelled` or `partially_completed` with the evidence saved.
- **Resume:** a new approval is required; the S3 policy checks re-run first.

## M. Completion conditions

Mandatory: `call_result` valid; S6 used an approved, policy-checked number; connection state truthful; recap supported by the transcript; charges reported; limitations disclosed; evidence references present.

- `completed` — call connected, goal met or clearly answered, result delivered.
- `partially_completed` — call connected but goal partly met, or the result could not be delivered, or the recording is missing.
- `blocked` — input, policy, provider readiness or approval prevented dialling.
- `failed` — the call was attempted but the provider failed, or a verification failure could not be corrected.
- `cancelled` — the user declined or approval timed out.

A not-connected call (`no_answer`, `busy`) is `completed` only for the act of trying if the user asked for one attempt; otherwise `partially_completed` with an offer to send a message instead.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome; `call_result`; step statuses S1–S11; verification results and failed checks; evidence references (approval ID, call ID, transcript and recording references); approvals and scope; side effects (call placed, charges) and reconciliation status; corrections performed; unresolved challenges; limitations; escalation state; learning candidate. Never include credentials, raw audio or full phone numbers in shared channels.

## O. Learning and revision

- **Outcome metrics:** connect rate, goal-met rate, average duration, cost per call, idle-timeout rate, recap-verification pass rate.
- **Evidence retained:** run record, approval record and transcript, per the retention policy.
- **Data minimisation:** metrics stored without transcript text; recordings deleted after the retention period.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}`.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review; no silent changes to trusted behaviour.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (sandbox number) | call connects, goal met, result delivered | UNVERIFIED on Skill Port; observed in the reference implementation |
| Missing goal | asks once, then `blocked` | TBD |
| Invalid number | rejected before policy checks | TBD |
| Ambiguous contact | asks which person | TBD |
| Missing capability (telephony binding absent) | `blocked` with a dependency message | TBD |
| Permission denied (`voice:call:create` missing) | `blocked`, no dial | TBD |
| Unauthorized scope (another tenant's contact) | denied | TBD |
| Tenant isolation | cannot read another tenant's calls | TBD |
| Quiet hours | `blocked` with the next window | TBD |
| Allowlist | non-allowlisted country `blocked` | TBD |
| Timeout on `place` | status read by key; no second dial | TBD |
| Duplicate / replay | same key returns the existing call | TBD |
| False success (provider OK, never answered) | `connected=false`, not `goal_met` | TBD |
| Failed verification (recap unsupported) | regenerated or downgraded | TBD |
| Correction budget exhausted | stops safely | TBD |
| Escalation (callee asks for a human) | ends politely, escalates | TBD |
| Cancellation (user declines at S5) | `cancelled`, nothing dialled | TBD |
| Compensation (user cuts a live call) | hang-up within 2 s, charges reported | TBD |
| Output schema and status | valid `call_result`, correct status | TBD |
| Compatibility / regression | passes on a new provider version | TBD |

All tests use sandbox or test numbers owned by the tester; never real customers. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, schema registry and approval UI — TBD (owner: publisher, with Skill Port).
- Per-call cost cap default — TBD (owner: operator).
- Retention period for recordings and transcripts — TBD (owner: operator, per local law).
- Recording disclosure rules differ by country; the runtime must enforce them.
- Caller-ID registration and telephony KYC are required in some countries before outbound calls work.
- Live latency depends on provider region (about 1 second voice-to-voice in the reference implementation).
- Languages are limited to those the configured STT/TTS providers support.
