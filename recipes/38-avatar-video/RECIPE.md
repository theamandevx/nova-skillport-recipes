# Avatar video

## A. Identity and ownership

- **Recipe ID:** `nova.content.avatar-video` (proposed until assigned by Skill Port)
- **Name:** Avatar video
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** design
- **Tags:** avatar-video, async-job, authenticated-callback, reconciler, durable-storage, cost-tracking
- **Summary:** The agent submits an approved script to an avatar-video provider as an asynchronous job, records it, and completes it when either the provider's authenticated callback arrives or a scheduled reconciler finds a job whose callback never arrived. Both paths re-fetch the authoritative status, download the file before the provider's pre-signed link expires, store it durably and leave exactly one cost record.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** An avatar-video provider with asynchronous generation, a status endpoint, avatar and voice catalogues and completion callbacks; a public HTTPS endpoint that can receive authenticated callbacks; a scheduler for the reconciler; durable object storage; a content database whose video-record function also writes the usage row; an owner notification channel. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — the core flow (asynchronous submit, completion by callback or reconciler, durable storage and the cost record) runs in Nova's reference implementation; not every control described here has evidence yet (Section P lists the tests still needed). Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.content.scripts` | Capability | Read a script and its approval state | low | UNVERIFIED |
| `nova.content.scripts.get` | Operation (read) | Return script text, status, hash and owner | low | UNVERIFIED |
| `nova.video.catalog` | Capability | Avatars and voices available to the user | low | UNVERIFIED |
| `nova.video.catalog.list_avatars` | Operation (read) | Return avatars with ownership (`stock`, `own`), supported engines and default voice | low | UNVERIFIED |
| `nova.video.catalog.list_voices` | Operation (read) | Return provider voices with languages | low | UNVERIFIED |
| `nova.video.account` | Capability | Provider readiness and credit balance | low | UNVERIFIED |
| `nova.video.account.get_readiness` | Operation (read) | Return `ready`, `reason`, credits remaining and reset time | low | UNVERIFIED |
| `nova.video.render` | Capability | Asynchronous avatar-video generation at the provider | medium | UNVERIFIED |
| `nova.video.render.submit` | Operation (create + paid external call) | Start one render job; returns the provider job ID with status `pending` | medium | UNVERIFIED |
| `nova.video.render.get_status` | Operation (read) | Authoritative status: `pending`, `processing`, `completed`, `failed`; pre-signed file URL (expires), thumbnail, duration, failure message | low | UNVERIFIED |
| `nova.video.completion` | Capability | Complete render jobs safely, whichever trigger arrives first | medium | UNVERIFIED |
| `nova.video.completion.receive_callback` | Operation (event) | Authenticate a provider callback and trigger `finalize` | medium | UNVERIFIED |
| `nova.video.completion.reconcile` | Operation (scheduled) | Sweep jobs still processing and trigger `finalize` | low | UNVERIFIED |
| `nova.video.completion.finalize` | Operation (update + create) | Re-fetch status, download, validate, store, advance the record | medium | UNVERIFIED |
| `nova.content.video` | Capability | Video job records, files and their usage rows | low | UNVERIFIED |
| `nova.content.video_records.add` | Operation (create) | Insert the job record **and** its usage row in one transaction | low | UNVERIFIED |
| `nova.content.video_records.update_status` | Operation (update) | Advance status monotonically, keyed by provider job ID | low | UNVERIFIED |
| `nova.content.video_records.list_active` | Operation (read) | Jobs in `processing` older than a threshold | low | UNVERIFIED |
| `nova.content.assets.upload` | Operation (create) | Store the video (and thumbnail) in a private video bucket | low | UNVERIFIED |
| `nova.notify.owner` | Capability | Deliver a message to the requesting user's own channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Tell the user the video is ready or failed | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.video.render.submit.text-voice` — the provider's own voice reads the script. Binding `binding://nova/video/submit` (UNVERIFIED); protocol: provider HTTPS API through a registered adapter; mode: async (job ID returned in about a second, file ready minutes later).
- `nova.video.render.submit.audio-driven` — the avatar lip-syncs to an existing voiceover file produced by `nova.content.voiceover`. Eligible only when that audio record belongs to the tenant and, for a cloned voice, its consent covers `video_narration`. Same binding and mode.
- **Engine selection:** each avatar lists the render engines it supports; the engine is an explicit parameter checked against that list. An unsupported engine is rejected; there is no silent switch to another engine, because engines differ in quality and credit cost.
- `nova.video.completion.receive_callback.default` — event-driven, authenticated (rules below).
- `nova.video.completion.reconcile.default` — scheduled every 5 minutes; selects jobs in `processing` for more than 2 minutes.
- Both completion triggers are **concurrent, not fallbacks**: either may run first, and both call the same idempotent `finalize`. The callback is optional; a deployment that cannot expose an authenticated public endpoint can run reconciler-only, accepting up to one sweep interval of extra latency.
- Every other operation supports only the default variant.

**Callback authentication (secure design):**

1. Prefer provider-signed callbacks: verify an HMAC signature over the raw body with `secret_ref://video-callback-signing` and reject timestamps older than 5 minutes.
2. Where the provider cannot sign, use a per-job capability token: at least 128 random bits, created at submit, stored only as a hash on the job record, compared in constant time, consumed on the first successful finalize and invalid after the job's maximum age.
3. Never log callback URLs, tokens or signatures; redact literal token values (not just URL paths) from logs, exported workflow definitions and source control; rotate the signing secret immediately if exposed.
4. The callback body is untrusted: it is only a trigger. `finalize` always re-reads the authoritative status from the provider.
5. Authentication failure → 401 and nothing processed; success → 202 at once, then process asynchronously; rate-limit the endpoint.

**Bindings:** `binding://nova/content/scripts`, `binding://nova/video/avatars`, `binding://nova/video/voices`, `binding://nova/video/account`, `binding://nova/video/submit`, `binding://nova/video/status`, `binding://nova/video/callback`, `binding://nova/video/reconcile`, `binding://nova/video/finalize`, `binding://nova/content/video-records`, `binding://nova/content/assets`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: submit, avatar and voice gateways plus a scheduled reconciler on the automation hub; a security-definer database function for video records whose `add` action also writes the usage row; a dedicated video bucket.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Script | `schema://nova/avatar-video/script@0.1.0` | the requesting user's tenant | read |
| Avatar, Voice | `schema://nova/avatar-video/avatar@0.1.0` | provider catalogue or the user's own avatars | read |
| Audio record (audio-driven only) | from `nova.content.voiceover` | the user's tenant | read |
| Video job | `schema://nova/avatar-video/video-job@0.1.0` | the user's tenant | create; update (status, file refs, token consumed) |
| Video asset | reference (storage key) | the user's private video bucket | create |
| Usage row | `schema://nova/avatar-video/usage-record@0.1.0` | the user's tenant ledger | create (inside `add`; adjustments append) |

**Authorization requirements (declared, never a grant):** `content:scripts:read`, `video:catalog:read`, `video:account:read`, `video:render:create`, `video:render:read`, `content:video:write`, `content:assets:write`, `notify:owner`; audio-driven only: `content:audio:read`. The callback endpoint and reconciler run under a service identity limited to `video:render:read`, `content:video:read` (active-job listing), `content:video:write` and `content:assets:write` for the job being finalized. Provider key: `secret_ref://video-provider`; storage: `secret_ref://object-storage`.

**Policy / trust:** risk class `medium` (paid async generation; a public inbound endpoint; a person's likeness when the avatar is the user's own). Requires a per-video cost cap, a maximum job age and the callback rules above.

## C. Intent and applicability

- **User problem:** The user wants a short presenter-style video of their avatar (or a stock avatar) reading a script, without filming.
- **Desired outcome:** One rendered video stored durably and linked to the request, with a truthful status and one cost record, even if the provider's callback is lost.
- **Trigger or entry conditions:** The user asks for an avatar video of a named script or post, or supplies script text and confirms it.
- **Applicable situations:** Short announcements, explainer clips, social video, narrated summaries.
- **Out-of-scope situations:** Creating a new avatar or digital twin (needs the provider's own consent flow); avatars of other real people; editing or publishing the video; long-form productions; scripts the user has not approved.
- **Required inputs:** `script_ref` or `script_text`, `avatar_id`, and `voice_id` or `audio_record_id` (see Section D).
- **Prerequisites:** A ready provider with credits; the avatar is stock or the user's own; storage and the video-record function available; the callback endpoint or the reconciler (or both) active.
- **Required capabilities:** `nova.content.scripts`, `nova.video.catalog`, `nova.video.account`, `nova.video.render`, `nova.video.completion`, `nova.content.video`, `nova.notify.owner` (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port; every binding is a dependency to resolve before activation.
- **Assumptions:** The core flow runs in the reference implementation. Assumed: the target provider offers equivalent status and callback features.
- **Ambiguity policy:** Ask once for a missing avatar or script; use documented defaults for aspect ratio (9:16) and resolution (720p); never pick another person's avatar.
- **Vedaxi task type:** none — this recipe is not one of the seven task types Vedaxi supports today.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `script_ref` / `script_text` | string; text 1–5,000 chars | one of the two | content store / user | — | Not approved → show and ask; too long → ask to split. |
| `avatar_id` | string | yes | user | operator default stock avatar | Not stock or own → `blocked`. |
| `voice_id` | string | for `text-voice` | user | the avatar's default voice | Unknown → offer voices. |
| `audio_record_id` | string | for `audio-driven` | `nova.content.voiceover` | — | Other tenant's or consent mismatch → `blocked`. |
| `engine` | string from the avatar's supported list | no | user | the avatar's first supported engine | Unsupported → rejected; no silent switch. |
| `aspect_ratio` | enum `9:16`, `16:9`, `1:1` | no | user | `9:16` | — |
| `resolution` | enum `720p`, `1080p` | no | user | `720p` | Not offered by the engine → ask. |
| `background_hex` | string `^#[0-9A-Fa-f]{6}$` | no | user | brand background | Invalid → default with notice. |
| `title` | string ≤ 80 chars | no | user | derived from the script | — |
| `max_cost_usd` | number > 0 | no | user | operator cap | Above cap → clamped. |

```json
{"$id":"schema://nova/avatar-video/request@0.1.0","type":"object","required":["avatar_id"],
 "oneOf":[{"required":["script_ref"]},{"required":["script_text"]}],
 "properties":{"script_ref":{"type":"string"},"script_text":{"type":"string","minLength":1,"maxLength":5000},
 "avatar_id":{"type":"string"},"voice_id":{"type":"string"},"audio_record_id":{"type":"string"},"engine":{"type":"string"},
 "aspect_ratio":{"enum":["9:16","16:9","1:1"]},"resolution":{"enum":["720p","1080p"]},
 "background_hex":{"type":"string","pattern":"^#[0-9A-Fa-f]{6}$"},"title":{"type":"string","maxLength":80},
 "max_cost_usd":{"type":"number","exclusiveMinimum":0}}}
```

```json
{"$id":"schema://nova/avatar-video/video-job@0.1.0","type":"object",
 "required":["job_id","tenant_id","client_key","status","created_at"],
 "properties":{"job_id":{"type":"string"},"tenant_id":{"type":"string"},"client_key":{"type":"string"},
 "provider_job_id":{"type":["string","null"]},"variant":{"enum":["text-voice","audio-driven"]},
 "status":{"enum":["submitting","processing","completed","failed"]},"callback_token_hash":{"type":["string","null"]},
 "token_consumed":{"type":"boolean"},"video_ref":{"type":["string","null"]},"thumbnail_ref":{"type":["string","null"]},
 "duration_s":{"type":["number","null"]},"failure_reason":{"type":["string","null"]},"completed_via":{"enum":["callback","reconciler",null]},
 "created_at":{"type":"string","format":"date-time"},"updated_at":{"type":"string","format":"date-time"}}}
```

Status order is `submitting` → `processing` → `completed` | `failed`; an update that would move backwards is ignored.

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `video_result` | object `schema://nova/avatar-video/video-result@0.1.0` | `job_id`, `status`, `job_status`, `cost` | the user's channel, the job record and the run record | internal; contains the user's likeness when the avatar is theirs | job record, stored file and usage row cross-checked (Section I) |

```json
{"$id":"schema://nova/avatar-video/video-result@0.1.0","type":"object","required":["job_id","status","job_status","cost"],
 "properties":{"job_id":{"type":"string"},"provider_job_id":{"type":["string","null"]},
 "status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},
 "job_status":{"enum":["submitting","processing","completed","failed"]},"completed_via":{"enum":["callback","reconciler",null]},
 "video_ref":{"type":["string","null"]},"thumbnail_ref":{"type":["string","null"]},"duration_s":{"type":["number","null"]},
 "aspect_ratio":{"type":"string"},"resolution":{"type":"string"},"engine":{"type":"string"},"ai_generated":{"type":"boolean","const":true},
 "cost":{"type":"object","required":["amount","currency","basis"],"properties":{"amount":{"type":"number","minimum":0},"currency":{"type":"string"},"basis":{"enum":["provider_reported","estimate"]},"credits":{"type":["number","null"]}}},
 "failure_reason":{"type":["string","null"]}}}
```

Supporting schemas:

```json
{"$id":"schema://nova/avatar-video/script@0.1.0","type":"object","required":["script_id","tenant_id","text","status","hash"],"properties":{"script_id":{"type":"string"},"tenant_id":{"type":"string"},"text":{"type":"string","minLength":1,"maxLength":5000},"status":{"enum":["draft","approved"]},"hash":{"type":"string"}}}
{"$id":"schema://nova/avatar-video/avatar@0.1.0","type":"object","required":["kind","id","name"],"properties":{"kind":{"enum":["avatar","voice"]},"id":{"type":"string"},"name":{"type":"string"},"ownership":{"enum":["stock","own"]},"engines":{"type":"array","items":{"type":"string"}},"default_voice_id":{"type":["string","null"]},"languages":{"type":"array","items":{"type":"string"}}}}
{"$id":"schema://nova/avatar-video/readiness@0.1.0","type":"object","required":["ready","reason"],"properties":{"ready":{"type":"boolean"},"reason":{"enum":["ready","invalid_credential","out_of_credits","provider_unavailable"]},"credits_remaining":{"type":["number","null"]},"resets_at":{"type":["string","null"],"format":"date-time"},"as_of":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/avatar-video/provider-status@0.1.0","type":"object","required":["provider_job_id","status","read_at"],"properties":{"provider_job_id":{"type":"string"},"status":{"enum":["pending","processing","completed","failed"]},"file_url_present":{"type":"boolean"},"file_url_expires_at":{"type":["string","null"],"format":"date-time"},"thumbnail_present":{"type":"boolean"},"duration_s":{"type":["number","null"]},"failure_message":{"type":["string","null"]},"credits_used":{"type":["number","null"]},"read_at":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/avatar-video/callback@0.1.0","type":"object","required":["correlation_id","received_at","auth_method"],"properties":{"correlation_id":{"type":"string"},"provider_job_id":{"type":["string","null"]},"received_at":{"type":"string","format":"date-time"},"auth_method":{"enum":["signature","job_token"]},"auth_result":{"enum":["valid","invalid"]},"claimed_status":{"type":["string","null"]}}}
{"$id":"schema://nova/avatar-video/sweep-summary@0.1.0","type":"object","required":["sweep_id","started_at","batch_size","outcomes"],"properties":{"sweep_id":{"type":"string"},"started_at":{"type":"string","format":"date-time"},"batch_size":{"type":"integer","minimum":0,"maximum":50},"outcomes":{"type":"object","required":["completed","failed","unchanged","timed_out","errors"],"properties":{"completed":{"type":"integer","minimum":0},"failed":{"type":"integer","minimum":0},"unchanged":{"type":"integer","minimum":0},"timed_out":{"type":"integer","minimum":0},"errors":{"type":"integer","minimum":0}}}}}
{"$id":"schema://nova/avatar-video/usage-record@0.1.0","type":"object","required":["usage_id","ts","provider","cost_usd","cost_basis","meta","idempotency_key"],"properties":{"usage_id":{"type":"string"},"ts":{"type":"string","format":"date-time"},"provider":{"type":"string"},"model":{"type":["string","null"]},"units":{"type":"object","properties":{"videos":{"type":"integer"},"credits":{"type":["number","null"]}}},"cost_usd":{"type":"number"},"cost_basis":{"enum":["provider_reported","estimate","price_table"]},"meta":{"type":"object","required":["job"],"properties":{"job":{"const":"content:avatar-video"},"run_id":{"type":"string"},"adjusts_usage_id":{"type":["string","null"]}},"additionalProperties":{"type":["string","null"]}},"idempotency_key":{"type":"string"}}}
{"$id":"schema://nova/avatar-video/delivery@0.1.0","type":"object","required":["message_id","status"],"properties":{"message_id":{"type":"string"},"channel":{"type":"string"},"status":{"enum":["delivered","queued","failed"]},"delivered_at":{"type":["string","null"],"format":"date-time"}}}
```

**Example (fictional)**

Input: `{"script_text":"Hi, I'm Jane from Acme Labs. This week we cut search time in half...","avatar_id":"avatar_own_01","voice_id":"voice_stock_12","aspect_ratio":"9:16","resolution":"720p"}`

Output: `{"job_id":"vid_0007","provider_job_id":"pv_5501","status":"completed","job_status":"completed","completed_via":"reconciler","video_ref":"asset://content-video/videos/pv_5501.mp4","thumbnail_ref":"asset://content-video/thumbs/pv_5501.jpg","duration_s":38.2,"aspect_ratio":"9:16","resolution":"720p","engine":"engine-standard","ai_generated":true,"cost":{"amount":0.30,"currency":"USD","basis":"estimate","credits":null},"failure_reason":null}` (price illustrative).

## E. Constraints and budgets

- **Hard constraints:** approved script only; avatar is stock or the user's own (created through the provider's consent flow); never another real person's likeness; engine explicitly supported by the avatar; estimate ≤ per-video cap; callbacks authenticated per Section B and treated only as triggers; status updates monotonic and keyed by provider job ID; the file is downloaded and stored before the pre-signed link expires; one usage row per job (adjustments appended, never edited); the video bucket is private by default; outputs marked `ai_generated: true`.
- **Soft constraints:** keep videos short; prefer the user's usual avatar and voice.
- **Resource and tenant scope:** the requesting user's scripts, avatars, audio records, video jobs, video bucket and ledger.
- **Data sources allowed:** the script, provider catalogues and status, the user's audio record (audio-driven).
- **External transfers permitted:** script text (or the referenced audio) to the video provider.
- **Time limit:** submit ≤ 30 s; interactive wait ≤ 30 minutes; maximum job age `TBD` (default 24 hours) before a stuck job is marked `failed`.
- **Cost or token budget:** per-video cap and daily video budget `TBD` (operator).
- **Maximum steps:** 11 (S1–S11).
- **Maximum retries and correction cycles:** submit: no automatic resubmit; status reads: 3 retries per sweep; download and upload: 2 retries each; correction cycles: 1 per step.
- **Rate limits and concurrency:** ≤ 2 jobs in flight per user (`TBD`); reconciler batch ≤ 50 jobs per sweep; one failing job never aborts the rest of the sweep.
- **Retention and logging constraints:** videos per the operator's retention policy (`TBD`); logs never contain provider keys, callback tokens, signatures or pre-signed URLs.
- **Prohibited actions:** resubmitting on an ambiguous outcome; acting on callback body fields without re-fetching; making the bucket public; deleting videos.
- **Stop conditions:** script not approved; avatar not allowed; provider not ready or out of credits; estimate over cap; job past maximum age.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Video job + usage row (one transaction) | S1–S4 passed | Allowed within cap | record readable; one usage row |
| Create | Video asset (+ thumbnail) | provider status `completed`; file validated | Allowed | stored object readable; size and duration match |
| Create | Usage adjustment row | provider reports actual credits or a refund | Allowed | references the original `usage_id` |
| Read | Script, catalogues, readiness, status, audio record | own tenant | Allowed | — |
| Update | Video job: `provider_job_id`, `status` (forward only), `video_ref`, `thumbnail_ref`, `duration_s`, `failure_reason`, `completed_via`, `token_consumed` | keyed by provider job ID | Allowed | read-back shows the new state |
| Delete | any | — | Not Allowed | — |
| External side effect | paid render job at the provider | estimate ≤ cap | Allowed within cap; Approval Required above | provider returns a job ID |
| External side effect | message to the requesting user | job finished or failed | Allowed | delivery receipt |

- **Idempotency:** `client_key` = hash(`run_id`, `script_hash`, `avatar_id`, voice or audio, engine, aspect, resolution) — unique on the job record, and sent to the provider as the correlation ID where supported. `update_status` only advances state, so callback and reconciler both firing is harmless.
- **Expected side effects:** provider credits consumed; one stored video; one job record and one usage row.
- **Compensation:** none for credits; a failed job stays recorded with its reason; a stored video can be left unlinked.
- **Irreversible:** credits consumed once the provider accepts the job.
- **Ambiguous outcomes:** if submit times out, list the provider's recent jobs and match the correlation ID before anything else; adopt a match; if none is found after two checks, mark the job `failed` (`submit_unknown`) and ask the user before any new submission.

## G. Autonomy and approval policy

- **Default autonomy level:** A2 — bounded autonomy within the cost cap; completion runs at A3 (scheduled, bounded recovery, escalates on maximum age).
- **Planning level:** A2.
- **Execution level:** A2 for submit within cap; A3 for callback and reconciler completion.
- **Verification level:** A2.
- **Recovery level:** A1 for any resubmission; A3 for retries of reads, downloads and uploads.
- **Approval required for:** approving a script supplied as text; a video above the cap; any resubmission after a failed or unknown submit.
- **Actions never permitted:** avatars of other real people; acting on unauthenticated callbacks; trusting callback fields without re-fetching; resubmitting automatically; publishing; deleting.
- **Conditions requiring clarification:** missing avatar or script; resolution not offered by the engine.
- **Conditions requiring escalation:** provider out of credits; job past maximum age; repeated authentication failures on the callback endpoint (possible probing).
- **Safe state after stop, cancellation or escalation:** no new submission; existing jobs keep being reconciled until they finish or reach maximum age; spend reported.

Approval covers one render job with the stated script, avatar, voice or audio, engine and estimate.

## H. Execution plan

**S1 — Validate the request**
1. Objective: a schema-valid request with one variant chosen.
2. Capability: none (local validation).
3. Input mapping: user message → `request@0.1.0`; `audio_record_id` present → `audio-driven`, else `text-voice`.
4. Dependencies: none.
5. Preconditions: explicit video request.
6. Action / side effects: validate; no side effects.
7. Autonomy: A2; no approval.
8. Timeout / retry: 5 s / none.
9. Idempotency: not applicable.
10. Postconditions: valid request and variant.
11. Verification: JSON Schema passes.
12. Failure routes: missing avatar or script → ask once → else `blocked`.
13. Escalation triggers: none.
14. Evidence: validated request.

**S2 — Resolve and approve the script**
1. Objective: an approved script and its hash.
2. Capability: `nova.content.scripts.get`; user interaction for supplied text.
3. Input mapping: `script_ref` or `script_text`.
4. Dependencies: S1.
5. Preconditions: request valid.
6. Action: read; if supplied or `draft`, show and ask "Render this exact script?".
7. Autonomy: A2 read; A0 approval.
8. Timeout / retry: 10 s, 2 retries; approval waits up to 30 minutes.
9. Idempotency: read-only.
10. Postconditions: approved text and `script_hash`.
11. Verification: length ≤ provider limit; owner = requester.
12. Failure routes: declined → `cancelled`; too long → ask to split → else `blocked`.
13. Escalation: none.
14. Evidence: script hash, approval ID if any.

**S3 — Readiness and catalogue checks**
1. Objective: confirm the provider, avatar, engine and voice or audio are usable.
2. Capability: `nova.video.account.get_readiness`; `nova.video.catalog.list_avatars` and `.list_voices`; audio record read (audio-driven).
3. Input mapping: `avatar_id`, `engine`, `voice_id` or `audio_record_id`, resolution.
4. Dependencies: S2.
5. Preconditions: script approved.
6. Action: read readiness and credits; confirm avatar ownership is `stock` or `own`; engine in the avatar's list; voice exists, or audio record owned with consent covering `video_narration` when its voice is cloned.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: all checks pass, or a named failure.
11. Verification: each check recorded with its reason.
12. Failure routes: any failure → `blocked` with the reason and valid alternatives (for example the engines this avatar supports).
13. Escalation: out of credits → tell the user.
14. Evidence: readiness snapshot, avatar and engine check.

**S4 — Estimate and confirm**
1. Objective: spend only within the approved scope.
2. Capability: user interaction (only when needed).
3. Input mapping: engine, resolution, estimated duration (script length), operator price or credit table.
4. Dependencies: S3.
5. Preconditions: checks passed.
6. Action: within cap → proceed and disclose; above → ask (Approve / Change / Cancel).
7. Autonomy: A2 within cap; A0 above.
8. Timeout / retry: 30 minutes, then `cancelled`.
9. Idempotency: approval binds `client_key`.
10. Postconditions: proceed flag or approval record.
11. Verification: approval references `client_key` and the estimate.
12. Failure routes: Change → S3; Cancel → `cancelled`.
13. Escalation: not applicable.
14. Evidence: estimate, approval ID.

**S5 — Create the job record**
1. Objective: claim the job before calling the provider.
2. Capability: `nova.content.video_records.add` (status `submitting`; usage row in the same transaction).
3. Input mapping: `client_key`, variant, settings, estimate, callback token hash (when tokens are used).
4. Dependencies: S4.
5. Preconditions: proceed flag.
6. Action / side effects: insert one record and one usage row (`cost_basis = estimate`); generate the callback token, keep only its hash.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: unique `client_key`; a second insert returns the existing record.
10. Postconditions: record in `submitting`.
11. Verification: read back the record and its usage row.
12. Failure routes: database unavailable → `blocked` (nothing submitted, nothing spent).
13. Escalation: none.
14. Evidence: `job_id`, `usage_id`.

**S6 — Submit to the provider**
1. Objective: start exactly one render job.
2. Capability: `nova.video.render.submit` (`text-voice` or `audio-driven`).
3. Input mapping: script or audio reference, avatar, voice, engine, aspect, resolution, background, title, callback URL with correlation ID (token in header where the provider allows, else in the URL).
4. Dependencies: S5.
5. Preconditions: record in `submitting`; approval valid.
6. Action / side effects: provider accepts the job; credits are committed.
7. Autonomy: A2 (covered by S4).
8. Timeout / retry: 30 s / **no automatic resubmit**.
9. Idempotency: `client_key` as correlation ID.
10. Postconditions: `provider_job_id` with status `pending`.
11. Verification: response contains a job ID.
12. Failure routes: timeout → ambiguous-outcome reconciliation (Section F); provider rejection → record `failed` with reason.
13. Escalation: unknown outcome after reconciliation → ask the user.
14. Evidence: provider job ID, request ID.

**S7 — Mark processing and wait**
1. Objective: the record reflects the provider job; the run waits a bounded time.
2. Capability: `nova.content.video_records.update_status`.
3. Input mapping: `job_id`, `provider_job_id`.
4. Dependencies: S6.
5. Preconditions: provider job ID known.
6. Action: advance to `processing`; wait up to 30 minutes for S10 to complete the job (via S8 or S9).
7. Autonomy: A2.
8. Timeout / retry: 10 s update, 2 retries; wait 30 minutes.
9. Idempotency: monotonic update keyed by provider job ID.
10. Postconditions: `processing`, or already finished.
11. Verification: read-back status.
12. Failure routes: wait expires → end the run `partially_completed` ("still rendering; you will be notified"); the job stays tracked.
13. Escalation: none.
14. Evidence: status timestamps.

**S8 — Callback trigger (event)**
1. Objective: complete a job promptly when the provider calls back.
2. Capability: `nova.video.completion.receive_callback.default`.
3. Input mapping: raw request (headers, body), correlation ID.
4. Dependencies: S6 (job exists).
5. Preconditions: callback endpoint active.
6. Action: authenticate (signature or token hash, constant time); on success reply 202 and enqueue S10; otherwise reply 401.
7. Autonomy: A3.
8. Timeout / retry: respond within 2 s; provider-side retries are absorbed by idempotency.
9. Idempotency: a consumed token or a finished job makes repeats no-ops.
10. Postconditions: S10 queued or the request rejected.
11. Verification: authentication result recorded (never the token).
12. Failure routes: authentication failure → 401, counted; unknown job → 401 without detail.
13. Escalation: more than 10 failures per hour (`TBD`) → alert the operator.
14. Evidence: callback received time, auth result, correlation ID.

**S9 — Reconciler trigger (scheduled)**
1. Objective: complete jobs whose callback never arrived.
2. Capability: `nova.video.completion.reconcile.default`; `nova.content.video_records.list_active`.
3. Input mapping: jobs in `processing` for more than 2 minutes, oldest first, ≤ 50.
4. Dependencies: S7.
5. Preconditions: scheduler active; single-flight lock held.
6. Action: run S10 for each job independently; mark jobs past maximum age `failed` (`timeout`).
7. Autonomy: A3.
8. Timeout / retry: every 5 minutes; per-job errors isolated.
9. Idempotency: same `finalize` keyed by provider job ID.
10. Postconditions: each job advanced, unchanged (still rendering) or failed.
11. Verification: sweep summary counts add up to the batch size.
12. Failure routes: provider down → leave jobs unchanged and retry next sweep.
13. Escalation: a job reaching maximum age → notify the user.
14. Evidence: sweep ID, per-job outcome.

**S10 — Finalize (shared by S8 and S9)**
1. Objective: turn a finished provider job into a stored, verified file.
2. Capability: `nova.video.render.get_status`; `nova.content.assets.upload`; `nova.content.video_records.update_status`.
3. Input mapping: `provider_job_id`.
4. Dependencies: S8 or S9.
5. Preconditions: record not already `completed` or `failed`.
6. Action / side effects: read the authoritative status; if `completed`, download the file at once (re-read status for a fresh link if expired), validate it (MP4 container, size > 0, duration within ± 2 s of the provider's value), upload video and thumbnail to `videos/<provider_job_id>.mp4`, then set `completed` with refs and `completed_via`; if `failed`, set `failed` with the provider's reason; otherwise leave unchanged.
7. Autonomy: A3.
8. Timeout / retry: status 15 s, 3 retries; download 120 s, 2 retries; upload 60 s, 2 retries.
9. Idempotency: monotonic update; hash-checked upload to a fixed path; token marked consumed.
10. Postconditions: record terminal with a durable file, or unchanged.
11. Verification: stored object readable and its size equals the downloaded size.
12. Failure routes: download or upload exhausted → leave `processing` for the next sweep; validation failure → `failed` (`invalid_file`).
13. Escalation: none here (maximum age handled by S9).
14. Evidence: provider status snapshot, file size, duration, `video_ref`.

**S11 — Verify and deliver**
1. Objective: tell the user the truthful outcome.
2. Capability: `nova.notify.owner.send`; optional usage adjustment.
3. Input mapping: job record, stored refs, provider-reported credits if any.
4. Dependencies: S10 reached a terminal state.
5. Preconditions: terminal state.
6. Action: run Section I checks; append a usage adjustment if actual credits differ from the estimate; send a private viewing link (time-limited) or the failure reason.
7. Autonomy: A3.
8. Timeout / retry: 15 s / 2 retries.
9. Idempotency: message key = `job_id` + terminal status.
10. Postconditions: user notified once.
11. Verification: delivery receipt.
12. Failure routes: delivery fails → result stays in the record; run status `partially_completed`.
13. Escalation: none.
14. Evidence: message ID, adjustment `usage_id` if any.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Input validity | request matches `request@0.1.0` | validator | valid | ask once, then `blocked` |
| Avatar allowed | ownership is `stock` or `own`; engine supported | catalogue | pass | `blocked` |
| Single submission | one provider job per `client_key` | job record | exactly one | reconcile; never resubmit |
| Callback authenticity | signature or token hash valid | S8 log | valid | 401; nothing processed |
| Authoritative state | terminal status written only after a provider status read | S10 snapshot | present | discard the update |
| Monotonic status | no backwards transition | record history | none | ignore stale update |
| Durable file | stored size = downloaded size; duration within ± 2 s | S10 | pass | retry or `failed` |
| One cost row | one usage row per job plus appended adjustments | ledger | pass | report the gap |
| Stuck jobs | no job in `processing` past maximum age | S9 | none | mark `failed`, notify |
| Output schema | `video_result` valid, `ai_generated = true` | validator | valid | fix fields; never fake values |

A provider callback saying "completed" proves nothing on its own; only a stored, validated file proves the video exists.

## J. Self-correction and recovery

- **Failure classes:** validation, unapproved script, avatar not allowed, provider not ready, timeout, ambiguous submit, transient provider error, expired file link, invalid file, storage failure, callback authentication failure, stuck job.
- **Permitted corrections:** re-ask for input once; reconcile an ambiguous submit by correlation ID; re-read status for a fresh file link; bounded retries of reads, downloads and uploads; leave a job for the next sweep.
- **Forbidden corrections:** automatic resubmission; trusting callback fields; loosening callback authentication to make deliveries pass; editing a usage row instead of appending an adjustment; switching engine or avatar without the user.
- **Maximum correction cycles:** 1 per step in the interactive run; per job, retries continue each sweep until maximum age.
- **Reverification required after correction:** yes — S10 validation after any re-download.
- **Stop conditions:** maximum age reached; hard constraint fails.
- **Recovery or compensation:** a failed job is reported with its reason; the user may approve a new submission.
- **What must be reported:** submission outcome, completion path (callback or reconciler), any adjustment to cost, failures and stuck jobs.

## K. Independent challenge

- **Assumptions to challenge:** the stored file is the job that was requested; the avatar is really the user's; the voice reads the approved script.
- **Alternative approach:** if renders keep failing for one engine, would a still image with a voiceover (recipes `nova.content.post-image` and `nova.content.voiceover`) serve the goal?
- **Contradictory evidence:** provider duration far from the expected script length; a callback for a job the record does not know.
- **Could mislead:** "completed" reported from a callback whose download failed; a cost estimate presented as billed.
- **Separate verifier:** the reconciler is itself an independent path — it re-reads provider state without relying on the callback; a duration plausibility check compares provider duration with script length (about 12–18 characters per second).
- **Findings requiring disclosure:** completion via reconciler (callback lost), cost basis `estimate`, partial runs still rendering.

## L. Human escalation

- **Triggers:** script not approved; estimate over cap; out of credits; ambiguous submit not resolved; job past maximum age; repeated callback authentication failures.
- **Why the agent cannot decide:** money, the user's likeness and a possible security probe.
- **Information presented:** script excerpt, avatar, engine, estimate, job record, provider status, failure reason, authentication failure counts.
- **Decision requested:** approve spend; approve a new submission; investigate the endpoint (operator).
- **Options and consequences:** for example, a new submission consumes credits again.
- **Safe behaviour while waiting:** no new submissions; reconciliation continues for existing jobs.
- **If no response:** the interactive run ends `cancelled` (nothing submitted) or `partially_completed` (job still tracked).
- **Resume:** S3 readiness re-runs before any new submission.

## M. Completion conditions

Mandatory: `video_result` valid; one provider job per approval; terminal state written only after an authoritative status read; durable file stored and validated for `completed`; one usage row (plus adjustments); limitations disclosed; evidence references present.

- `completed` — the job finished, the file is stored and validated, the cost is recorded and the user was notified.
- `partially_completed` — the job was submitted and is still rendering when the interactive wait ended, or the file is stored but delivery failed.
- `blocked` — input, approval, avatar, readiness or budget prevented submission.
- `failed` — the provider rejected or failed the job, the file was invalid, the job passed maximum age, or a submit outcome stayed unknown.
- `cancelled` — the user declined or approval timed out.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome; `video_result`; step statuses S1–S11 (S8 or S9, whichever completed the job); verification results and failed checks; evidence references (approval ID, `job_id`, provider job ID, `video_ref`, `usage_id`, sweep ID); approvals and scope; side effects (credits, stored file) and reconciliation status; corrections; unresolved challenges; limitations; escalation state; learning candidate. Never include provider keys, callback tokens, signatures or pre-signed URLs.

## O. Learning and revision

- **Outcome metrics:** render success rate per engine, time to completion, share completed via callback vs reconciler, stuck-job rate, cost per video, estimate-vs-actual delta.
- **Evidence retained:** job records, sweep summaries, usage rows.
- **Data minimisation:** metrics without script text; callback logs without tokens or bodies.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}`.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review; callback authentication can never be weakened by an automated revision.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path via callback (stock avatar, test script) | job completes, file stored, one usage row, user notified | UNVERIFIED on Skill Port; observed in the reference implementation |
| Happy path via reconciler (callback disabled) | completed within one sweep after render | UNVERIFIED on Skill Port; observed in the reference implementation |
| Missing input (no avatar) | asks once, then `blocked` | TBD |
| Invalid input (unsupported engine, script over limit) | rejected; alternatives listed | TBD |
| Missing capability (submit binding absent) | `blocked` with a dependency message | TBD |
| Permission denied (`video:render:create` missing) | `blocked`, nothing submitted | TBD |
| Unauthorized scope (another tenant's avatar or audio record) | denied | TBD |
| Tenant isolation | callback for tenant A cannot update tenant B's job | TBD |
| Forged callback (bad signature or token) | 401; nothing processed | TBD |
| Timeout on submit | correlation-ID reconciliation; no second job | TBD |
| Duplicate / replay (callback delivered twice; callback and reconciler together) | one terminal update, one file, one usage row | TBD |
| Expired file link | fresh link fetched; file stored | TBD |
| False success (callback says completed, provider status says processing) | record stays `processing` | TBD |
| Failed verification (corrupt or zero-byte file) | `failed` (`invalid_file`) | TBD |
| One bad job in a sweep | other jobs still finalized | TBD |
| Correction budget exhausted / stuck job past maximum age | `failed` (`timeout`), user notified | TBD |
| Escalation (out of credits) | user told; nothing submitted | TBD |
| Cancellation (user declines at S4) | `cancelled`; no job record | TBD |
| Compensation (failed job with credit refund reported) | adjustment row appended | TBD |
| Output schema and status | valid `video_result`, correct status | TBD |
| Compatibility / regression (provider API version change) | status mapping and callback format still valid | TBD |

All tests use a sandbox tenant, stock avatars and test scripts; never real customers' likenesses. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, schema registry, approval UI and inbound callback hosting — TBD (owner: publisher, with Skill Port).
- Per-video cap, daily budget, credit-to-currency table and maximum job age — TBD (owner: operator).
- Whether the provider signs callbacks; if not, the per-job token scheme applies.
- Whether failed jobs consume or refund credits — TBD per provider.
- Creating the user's own avatar requires the provider's consent flow, outside this recipe.
- Retention of videos — TBD (owner: operator).
