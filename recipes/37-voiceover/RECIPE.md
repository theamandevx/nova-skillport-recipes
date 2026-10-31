# Voiceover

## A. Identity and ownership

- **Recipe ID:** `nova.content.voiceover` (proposed until assigned by Skill Port)
- **Name:** Voiceover
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** design
- **Tags:** text-to-speech, voiceover, audio, voice-selection, quota-check, cost-tracking, voice-cloning-consent
- **Summary:** From an approved script, the agent checks the text-to-speech provider's readiness and remaining character quota, confirms the voice and the estimated cost, generates the audio, stores the file and records characters and cost exactly once. A cloned voice is never created or used without a recorded, unrevoked consent from the voice's owner.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** A text-to-speech provider with a voice catalogue, a quota endpoint and a generation history; durable object storage; a content database whose audio-record function also writes the usage row; a consent store (for cloned voices); an owner notification channel. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — the core flow (synthesis from an approved script with a chosen voice, storage, and the audio record with its cost row) runs in Nova's reference implementation; not every control described here has evidence yet (Section P lists the tests still needed). Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.content.scripts` | Capability | Read a script and its approval state | low | UNVERIFIED |
| `nova.content.scripts.get` | Operation (read) | Return script text, status (`draft`, `approved`), hash and owner | low | UNVERIFIED |
| `nova.voice.library` | Capability | Voices available to the user, including cloned voices | medium | UNVERIFIED |
| `nova.voice.library.list_voices` | Operation (read) | Return voices with language, category (`stock`, `cloned`) and `consent_ref` for cloned voices | low | UNVERIFIED |
| `nova.voice.library.clone_voice` | Operation (create + external transfer of voice samples) | Create a cloned voice from the owner's samples; consent-gated | high | UNVERIFIED |
| `nova.voice.library.set_voice_status` | Operation (update) | Disable a cloned voice when its consent is revoked or expires | medium | UNVERIFIED |
| `nova.consent.voice` | Capability | Consent records for cloned voices (written only by the owner's verified consent flow) | high | UNVERIFIED |
| `nova.consent.voice.get` | Operation (read) | Return a consent record: owner, scope, purposes, expiry, revocation | low | UNVERIFIED |
| `nova.voice.account` | Capability | Provider readiness and character quota | low | UNVERIFIED |
| `nova.voice.account.get_quota` | Operation (read) | Return `ready`, `reason`, characters used, limit and reset time | low | UNVERIFIED |
| `nova.voice.tts` | Capability | Generate speech audio from text | medium | UNVERIFIED |
| `nova.voice.tts.synthesize` | Operation (create + paid external call) | Generate one audio file from the script with the chosen voice | medium | UNVERIFIED |
| `nova.voice.history` | Capability | Past generations | low | UNVERIFIED |
| `nova.voice.history.find` | Operation (read) | Find a past generation by request hash | low | UNVERIFIED |
| `nova.content.audio` | Capability | Store audio and write its record and usage row | low | UNVERIFIED |
| `nova.content.assets.upload` | Operation (create) | Store the audio file under a content-hash path | low | UNVERIFIED |
| `nova.content.audio_records.add` | Operation (create) | Insert the audio record **and** its usage row in one database transaction | low | UNVERIFIED |
| `nova.speech.verify` | Capability (independent verifier) | Independent fidelity check of generated speech | low | UNVERIFIED |
| `nova.speech.transcribe` | Operation (read) | Transcribe the audio back for comparison with the script | low | UNVERIFIED |
| `nova.notify.owner` | Capability | Deliver a message to the requesting user's own channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Send the result to the user | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.voice.library.list_voices.live` — the provider's live voice listing; read-only. Selected first.
- `nova.voice.library.list_voices.allowlist` — the operator's configured voice catalogue (voice IDs validated when configured); read-only. Eligible only when the live listing reports `missing_permission` or `provider_unavailable`. Same constraints and scopes as `live`; its use is disclosed ("voice list from operator catalogue"). Cloned voices are never resolved from the allowlist — they always need a live consent check.
- `nova.voice.tts.synthesize.default` — the provider's multilingual model, MP3 44.1 kHz 128 kbps by default. Model and output format are user-visible parameters from the provider's declared list, never a silent substitution. Binding `binding://nova/voice/tts` (UNVERIFIED); protocol: provider API through the automation hub's webhook gateway; mode: sync.
- `nova.voice.library.clone_voice.default` — only the default variant; always requires a valid consent record and an explicit approval (Section G).
- Every other operation supports only the default variant.

**Provider readiness reasons (returned by `get_quota` and `list_voices`):** `ready`, `missing_permission` (the key lacks a scope such as voice listing or account read), `invalid_credential`, `payment_required`, `quota_exhausted`, `provider_unavailable`. Readiness is per operation: a key can synthesize while lacking voice-listing or account-read permission.

**Bindings:** `binding://nova/content/scripts`, `binding://nova/voice/voices`, `binding://nova/voice/clone`, `binding://nova/voice/voice-status`, `binding://nova/consent/voice`, `binding://nova/voice/quota`, `binding://nova/voice/tts`, `binding://nova/voice/history`, `binding://nova/content/assets`, `binding://nova/content/audio-records`, `binding://nova/speech/transcribe`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: header-authenticated webhook gateways on the automation hub (synthesis, voices, history, quota), an audio bucket, and a security-definer database function whose `add` action writes the audio row and the usage row together, so every caller is counted exactly once.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Script | `schema://nova/voiceover/script@0.1.0` | the requesting user's tenant | read |
| Voice | `schema://nova/voiceover/voice@0.1.0` | provider catalogue or the user's cloned voices | read; create (clone, consent); update (disable clone) |
| Consent record | `schema://nova/voiceover/consent@0.1.0` | the voice owner | read only |
| Audio asset | reference (storage key) | the user's audio bucket | create |
| Audio record | `schema://nova/voiceover/audio-record@0.1.0` | the user's tenant | create |
| Usage row | `schema://nova/voiceover/usage-record@0.1.0` | the user's tenant ledger | create (inside `audio_records.add`) |

**Authorization requirements (declared, never a grant):** `content:scripts:read`, `voice:library:read`, `voice:account:read`, `voice:tts:create`, `voice:history:read`, `content:assets:write`, `content:audio:write`, `notify:owner`; only with a clone request: `voice:clone:create`, `consent:voice:read`; only on revocation handling: `voice:library:update`. Provider key: `secret_ref://tts-provider`; storage: `secret_ref://object-storage`.

**Policy / trust:** risk class `medium` for synthesis with stock or already-consented voices; `high` for cloning (biometric data, impersonation risk). Requires a per-run character cap and cost cap; cloning requires consent plus action-specific approval.

## C. Intent and applicability

- **User problem:** A post, video or announcement needs a clean voiceover read in a chosen voice, without recording it.
- **Desired outcome:** One audio file that speaks the approved script word for word, stored durably, with characters and cost recorded once.
- **Trigger or entry conditions:** The user asks for a voiceover of a named script or post, or supplies script text and confirms it.
- **Applicable situations:** Post voiceovers, short explainers, narration to drive an avatar video (recipe `nova.content.avatar-video`), audio previews.
- **Out-of-scope situations:** Live or real-time speech; phone calls; imitating a real person without consent; cloning public figures or voices taken from public or scraped audio; rewriting the script (that is a drafting recipe); music or sound effects.
- **Required inputs:** `script_ref` or `script_text`, `voice_id` (see Section D).
- **Prerequisites:** A ready text-to-speech provider; quota readable or an operator rule allowing an unverified quota; storage and audio-record function available.
- **Required capabilities:** `nova.content.scripts`, `nova.voice.library`, `nova.voice.account`, `nova.voice.tts`, `nova.voice.history`, `nova.content.assets.upload`, `nova.content.audio_records.add`, `nova.notify.owner.send`; for cloning also `nova.consent.voice` (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port. The transcribe-back verifier needs an independent speech-to-text binding.
- **Assumptions:** The core flow runs in the reference implementation. Assumed: the provider counts billable characters the way the estimate does (`TBD` per provider).
- **Ambiguity policy:** Ask. If the script or voice is unclear, ask once; never pick a cloned voice by default; never alter the script.
- **Vedaxi task type:** none — this recipe is not one of the seven task types Vedaxi supports today.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `script_ref` | string | one of `script_ref` / `script_text` | content store | — | Not approved → show it and ask for approval. |
| `script_text` | string, 1–5,000 chars | one of the two | user | — | Longer than the provider limit → ask the user to split; never truncate silently. |
| `voice_id` | string | yes | user | operator default stock voice | Unknown → offer ready voices; cloned without valid consent → `blocked`. |
| `model` | string from the provider's declared list | no | user | provider default | Unknown → reject; no substitution. |
| `output_format` | enum `mp3_44100_128`, `mp3_22050_64`, `wav_44100` | no | user | `mp3_44100_128` | Unsupported → reject. |
| `language` | BCP 47 tag | no | user | detected from the script | Voice does not support it → ask. |
| `link_to` | object `{type, id}` (type `post` or `video`) | no | user | — | Other tenant's object → rejected. |
| `max_cost_usd` | number > 0 | no | user | operator cap | Above the cap → clamped. |
| `clone_request` | object (below) | no | user | — | Present → clone path S5; consent required. |

```json
{"$id":"schema://nova/voiceover/request@0.1.0","type":"object","required":["voice_id"],
 "oneOf":[{"required":["script_ref"]},{"required":["script_text"]}],
 "properties":{"script_ref":{"type":"string"},"script_text":{"type":"string","minLength":1,"maxLength":5000},
 "voice_id":{"type":"string"},"model":{"type":"string"},"output_format":{"enum":["mp3_44100_128","mp3_22050_64","wav_44100"]},
 "language":{"type":"string"},"link_to":{"type":"object","required":["type","id"],"properties":{"type":{"enum":["post","video"]},"id":{"type":"string"}}},
 "max_cost_usd":{"type":"number","exclusiveMinimum":0},
 "clone_request":{"type":"object","required":["voice_name","sample_refs","consent_ref"],"properties":{"voice_name":{"type":"string","maxLength":60},
  "sample_refs":{"type":"array","minItems":1,"maxItems":25,"items":{"type":"string"}},"consent_ref":{"type":"string"}}}}}
```

```json
{"$id":"schema://nova/voiceover/consent@0.1.0","type":"object",
 "required":["consent_id","voice_owner","granted_by_owner","scope","statement_ref","granted_at"],
 "properties":{"consent_id":{"type":"string"},"voice_owner":{"type":"object","required":["display_name","identity_verified"],
  "properties":{"display_name":{"type":"string"},"identity_verified":{"type":"boolean"},"verification_method":{"type":"string"}}},
 "granted_by_owner":{"type":"boolean","const":true},"scope":{"type":"object","required":["cloning","purposes"],
  "properties":{"cloning":{"type":"boolean"},"purposes":{"type":"array","items":{"enum":["post_voiceover","video_narration","internal_preview"]}}}},
 "statement_ref":{"type":"string"},"granted_at":{"type":"string","format":"date-time"},
 "expires_at":{"type":["string","null"],"format":"date-time"},"revoked_at":{"type":["string","null"],"format":"date-time"}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `voiceover_result` | object `schema://nova/voiceover/voiceover-result@0.1.0` | `status`, `voice_id`, `characters`, `audio_ref`, `cost` | the user's channel, the audio record and the run record | internal; the audio is AI-generated speech | file, duration, record and usage row cross-checked (Section I) |

```json
{"$id":"schema://nova/voiceover/voiceover-result@0.1.0","type":"object","required":["status","voice_id","characters","audio_ref","cost"],
 "properties":{"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},
 "audio_record_id":{"type":["string","null"]},"voice_id":{"type":"string"},"voice_category":{"enum":["stock","cloned"]},
 "consent_ref":{"type":["string","null"]},"model":{"type":"string"},"output_format":{"type":"string"},
 "characters":{"type":"integer","minimum":0},"duration_s":{"type":["number","null"]},"audio_ref":{"type":["string","null"]},
 "content_hash":{"type":["string","null"]},"script_hash":{"type":"string"},"ai_generated":{"type":"boolean","const":true},
 "quota":{"type":"object","properties":{"remaining_after":{"type":["integer","null"]},"verified":{"type":"boolean"}}},
 "cost":{"type":"object","required":["amount","currency","basis"],"properties":{"amount":{"type":"number","minimum":0},"currency":{"type":"string"},"basis":{"enum":["provider_reported","estimate"]}}},
 "reused_existing":{"type":"boolean"}}}
```

Supporting schemas:

```json
{"$id":"schema://nova/voiceover/script@0.1.0","type":"object","required":["script_id","tenant_id","text","status","hash"],"properties":{"script_id":{"type":"string"},"tenant_id":{"type":"string"},"text":{"type":"string","minLength":1,"maxLength":5000},"status":{"enum":["draft","approved"]},"hash":{"type":"string"}}}
{"$id":"schema://nova/voiceover/voice@0.1.0","type":"object","required":["voice_id","name","category","status"],"properties":{"voice_id":{"type":"string"},"name":{"type":"string"},"languages":{"type":"array","items":{"type":"string"}},"category":{"enum":["stock","cloned"]},"consent_ref":{"type":["string","null"]},"status":{"enum":["active","disabled"]},"source_variant":{"enum":["live","allowlist"]}}}
{"$id":"schema://nova/voiceover/quota@0.1.0","type":"object","required":["ready","reason"],"properties":{"ready":{"type":"boolean"},"reason":{"enum":["ready","missing_permission","invalid_credential","payment_required","quota_exhausted","provider_unavailable"]},"characters_used":{"type":["integer","null"],"minimum":0},"characters_limit":{"type":["integer","null"],"minimum":0},"resets_at":{"type":["string","null"],"format":"date-time"},"as_of":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/voiceover/audio-record@0.1.0","type":"object","required":["audio_record_id","tenant_id","script_hash","voice_id","characters","created_at"],"properties":{"audio_record_id":{"type":"string"},"tenant_id":{"type":"string"},"request_hash":{"type":"string"},"script_hash":{"type":"string"},"voice_id":{"type":"string"},"model":{"type":"string"},"characters":{"type":"integer","minimum":1},"duration_s":{"type":["number","null"]},"audio_ref":{"type":["string","null"]},"content_hash":{"type":["string","null"]},"usage_id":{"type":["string","null"]},"created_at":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/voiceover/usage-record@0.1.0","type":"object","required":["usage_id","ts","provider","cost_usd","cost_basis","meta","idempotency_key"],"properties":{"usage_id":{"type":"string"},"ts":{"type":"string","format":"date-time"},"provider":{"type":"string"},"model":{"type":["string","null"]},"units":{"type":"object","properties":{"characters":{"type":"integer","minimum":0}}},"cost_usd":{"type":"number"},"cost_basis":{"enum":["provider_reported","estimate","price_table"]},"meta":{"type":"object","required":["job"],"properties":{"job":{"const":"content:voiceover"},"run_id":{"type":"string"},"adjusts_usage_id":{"type":["string","null"]}},"additionalProperties":{"type":["string","null"]}},"idempotency_key":{"type":"string"}}}
{"$id":"schema://nova/voiceover/fidelity@0.1.0","type":"object","required":["audio_record_id","verifier_status"],"properties":{"audio_record_id":{"type":"string"},"verifier_status":{"enum":["pass","fail","not_run"]},"wer":{"type":["number","null"],"minimum":0},"threshold":{"type":["number","null"]},"quota_delta_chars":{"type":["integer","null"]}}}
{"$id":"schema://nova/voiceover/delivery@0.1.0","type":"object","required":["message_id","status"],"properties":{"message_id":{"type":"string"},"channel":{"type":"string"},"status":{"enum":["delivered","queued","failed"]},"delivered_at":{"type":["string","null"],"format":"date-time"}}}
```

**Example (fictional)**

Input: `{"script_text":"Three things changed in our release this week. First, search is twice as fast...","voice_id":"voice_stock_07","link_to":{"type":"post","id":"post_0042"}}`

Output: `{"status":"completed","audio_record_id":"aud_0311","voice_id":"voice_stock_07","voice_category":"stock","consent_ref":null,"model":"tts-multilingual","output_format":"mp3_44100_128","characters":412,"duration_s":27.4,"audio_ref":"asset://content-audio/voiceovers/aud_0311.mp3","content_hash":"sha256:7c2a","script_hash":"sha256:51de","ai_generated":true,"quota":{"remaining_after":88140,"verified":true},"cost":{"amount":0.12,"currency":"USD","basis":"estimate"},"reused_existing":false}` (hashes shortened; price illustrative, not a current vendor price).

## E. Constraints and budgets

- **Hard constraints:** the script is approved and spoken word for word (whitespace normalisation only); the voice is a stock voice or a cloned voice whose consent record is valid, owner-granted, unrevoked, unexpired and covers the purpose; cloning only with a consent record and an explicit approval; never clone from public, scraped or third-party audio, or a public figure's voice; never use a voice to impersonate a person; characters ≤ per-run cap; estimate ≤ cost cap; quota verified before synthesis (unless the operator explicitly allows unverified quota, then the per-run character cap still applies); every output is marked `ai_generated: true`.
- **Soft constraints:** prefer the user's usual voice; prefer the default format; keep generations short.
- **Resource and tenant scope:** the requesting user's scripts, voices, audio bucket and ledger.
- **Data sources allowed:** the script, the voice catalogue, the consent store, the provider's quota and history.
- **External transfers permitted:** script text to the TTS provider; for cloning only, the owner's consented samples to the same provider.
- **Time limit:** synthesis ≤ 120 s; whole run ≤ 10 minutes (cloning may take longer — provider dependent, `TBD`).
- **Cost or token budget:** per-run character cap and cost cap `TBD` (operator).
- **Maximum steps:** 12 (S1–S12).
- **Maximum retries and correction cycles:** synthesis: 1 retry on a transient error after a history check; reads: 2 retries; correction cycles: 1 per step.
- **Rate limits and concurrency:** 1 synthesis in flight per user; provider rate limits apply.
- **Retention and logging constraints:** audio kept per the operator's retention policy (`TBD`); consent statements kept as long as the cloned voice exists plus the legal retention period (`TBD`); logs never contain provider keys, full scripts or voice samples.
- **Prohibited actions:** editing the script; creating or editing consent records; re-enabling a disabled cloned voice; deleting audio or voices.
- **Stop conditions:** script not approved; provider not ready; quota insufficient or unreadable (without the operator rule); consent invalid; estimate over cap.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Audio asset + audio record + usage row | S9 passed | Allowed within the pre-approved cap | record and usage row read back; hashes match |
| Create | Cloned voice | valid consent covering cloning and purpose | Approval Required (S5) | voice appears in `list_voices` with `consent_ref` |
| Read | Script, voices, consent record, quota, history | own tenant; consent read only for the referenced record | Allowed | — |
| Update | Cloned voice `status` → `disabled` | consent revoked or expired | Allowed (protective) | voice no longer selectable |
| Delete | any | — | Not Allowed (provider-side clone deletion is an owner action outside this recipe) | — |
| External side effect | paid synthesis call | approved script, ready provider, quota, cap | Allowed within cap; Approval Required above cap | audio returned |
| External side effect | voice samples sent to the provider | clone path only | Approval Required (S5) | provider confirms the clone |
| External side effect | result message to the requesting user | run finished | Allowed | delivery receipt |

- **Idempotency:** request hash = hash(`script_hash`, `voice_id`, `model`, `output_format`, voice settings). S7 looks the hash up in history and in audio records; the audio record has a unique key on (`tenant_id`, request hash, `run_id`).
- **Expected side effects:** characters consumed from quota; provider charges; one audio object, one audio record, one usage row.
- **Compensation:** none needed for internal records; a generated file can be left unlinked. A clone can be disabled immediately; provider-side deletion is an owner action.
- **Irreversible:** characters consumed and charges; samples already sent to the provider for cloning.
- **Ambiguous outcomes:** if synthesis times out, query history by request hash before retrying; if the provider shows a completed generation, fetch it instead of generating again.

## G. Autonomy and approval policy

- **Default autonomy level:** A2 for synthesis with an approved script and a stock or consented voice, within caps; A1 for cloning.
- **Planning level:** A2.
- **Execution level:** A2 within caps; A0/A1 for cloning (owner consent plus user approval).
- **Verification level:** A2.
- **Recovery level:** A1.
- **Approval required for:** approving a script supplied as text; synthesis above the caps; every clone; reusing an existing audio file instead of generating (user chooses).
- **Actions never permitted:** cloning without a valid owner-granted consent; creating consent on someone's behalf; impersonation; changing script words; using a cloned voice outside its consented purposes; deleting voices or audio.
- **Conditions requiring clarification:** ambiguous script or voice; language not supported by the voice.
- **Conditions requiring escalation:** consent missing, expired or revoked; provider `payment_required` or `invalid_credential`; quota exhausted.
- **Safe state after stop, cancellation or escalation:** no synthesis or clone in progress; nothing deleted; characters and charges so far reported.

Approval covers one generation (or one clone) with the stated script, voice and estimate.

## H. Execution plan

**S1 — Validate the request**
1. Objective: a schema-valid request.
2. Capability: none (local validation).
3. Input mapping: user message → `request@0.1.0`.
4. Dependencies: none.
5. Preconditions: explicit voiceover request.
6. Action / side effects: validate; no side effects.
7. Autonomy: A2; no approval.
8. Timeout / retry: 5 s / none.
9. Idempotency: not applicable.
10. Postconditions: valid request.
11. Verification: JSON Schema passes.
12. Failure routes: missing script or voice → ask once → else `blocked`.
13. Escalation triggers: none.
14. Evidence: validated request.

**S2 — Resolve and approve the script**
1. Objective: an approved script and its hash.
2. Capability: `nova.content.scripts.get` (for `script_ref`); user interaction (for `script_text`).
3. Input mapping: `script_ref` or `script_text`.
4. Dependencies: S1.
5. Preconditions: request valid.
6. Action: read the script; if text was supplied or the script is `draft`, show it and ask "Voice this exact script?".
7. Autonomy: A2 read; A0 approval.
8. Timeout / retry: 10 s read, 2 retries; approval waits up to 30 minutes.
9. Idempotency: read-only.
10. Postconditions: approved text, `script_hash`, character count.
11. Verification: character count ≤ provider per-request limit; owner = requester.
12. Failure routes: not approved → `cancelled`; too long → ask to split → else `blocked`.
13. Escalation: none.
14. Evidence: script hash, approval ID if any.

**S3 — Provider readiness and quota**
1. Objective: confirm the provider can serve this many characters now.
2. Capability: `nova.voice.account.get_quota`.
3. Input mapping: character count.
4. Dependencies: S2.
5. Preconditions: script approved.
6. Action: read `ready`, `reason`, used, limit and reset time; compute remaining after this run.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 1 retry.
9. Idempotency: read-only.
10. Postconditions: quota verified, or a named reason.
11. Verification: remaining ≥ characters needed.
12. Failure routes: `quota_exhausted` → `blocked` with reset time; `payment_required` or `invalid_credential` → `blocked` and escalate; `missing_permission` → continue only if the operator set `allow_unverified_quota` and characters ≤ per-run cap, disclosed; else `blocked`.
13. Escalation: billing or credential problems → tell the user.
14. Evidence: quota snapshot with timestamp.

**S4 — Resolve the voice and check consent**
1. Objective: one valid voice for this purpose.
2. Capability: `nova.voice.library.list_voices` (`live`, then `allowlist`); `nova.consent.voice.get` for cloned voices.
3. Input mapping: `voice_id`, language, purpose (from `link_to`).
4. Dependencies: S3.
5. Preconditions: provider ready.
6. Action: confirm the voice exists, is `active` and supports the language; for a cloned voice read its consent record.
7. Autonomy: A2.
8. Timeout / retry: 10 s per variant / 2 retries.
9. Idempotency: read-only.
10. Postconditions: voice resolved; consent valid when cloned.
11. Verification: consent `granted_by_owner = true`, `scope.purposes` contains the purpose, not expired, not revoked.
12. Failure routes: unknown voice → offer ready voices; consent invalid → `blocked`, and if revoked or expired run `set_voice_status` → `disabled`.
13. Escalation: consent problems → tell the user (the owner must renew consent themselves).
14. Evidence: voice ID, variant used, consent ID and check result.

**S5 — Clone a voice (only when `clone_request` is present)**
1. Objective: create a cloned voice only with owner consent and user approval.
2. Capability: `nova.consent.voice.get`; `nova.voice.library.clone_voice.default`.
3. Input mapping: `clone_request` (`voice_name`, `sample_refs`, `consent_ref`).
4. Dependencies: S3.
5. Preconditions: consent record valid with `scope.cloning = true`, owner identity verified, purpose covered; samples belong to the user's tenant.
6. Action / side effects: ask "Create a cloned voice of <owner display name> from 3 samples under consent C-17?"; on approval send the samples to the provider.
7. Autonomy: A0 approval, then A1.
8. Timeout / retry: provider dependent (`TBD`) / no automatic retry.
9. Idempotency: key = hash(`consent_ref`, sample hashes); an existing clone for the same key is returned.
10. Postconditions: new `voice_id` with `consent_ref` attached.
11. Verification: voice appears in `list_voices` as `cloned` with the consent reference.
12. Failure routes: consent invalid → `blocked`; approval declined → `cancelled`; provider rejects → `failed`.
13. Escalation: any doubt that the speaker in the samples is the consenting owner → stop and ask.
14. Evidence: consent ID, approval ID, sample hashes (not audio), new voice ID.

**S6 — Estimate and confirm**
1. Objective: spend only within the approved scope.
2. Capability: user interaction (only when needed).
3. Input mapping: characters, voice, model, format, estimate from the operator's price table.
4. Dependencies: S4 (and S5 if run).
5. Preconditions: voice valid.
6. Action: within caps → proceed and disclose; above → ask (Approve / Change / Cancel).
7. Autonomy: A2 within caps; A0 above.
8. Timeout / retry: 30 minutes, then `cancelled`.
9. Idempotency: approval binds the request hash.
10. Postconditions: proceed flag or approval record.
11. Verification: approval references the request hash.
12. Failure routes: Change → S4; Cancel → `cancelled`.
13. Escalation: not applicable.
14. Evidence: estimate, approval ID.

**S7 — Duplicate check**
1. Objective: never pay twice for the same audio unknowingly.
2. Capability: `nova.voice.history.find`; audio records read.
3. Input mapping: request hash.
4. Dependencies: S6.
5. Preconditions: proceed flag.
6. Action: look for an existing generation with the same hash; if found, offer reuse.
7. Autonomy: A2; reuse needs the user's yes.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: reuse chosen or new generation needed.
11. Verification: hash equality.
12. Failure routes: history unavailable → proceed and disclose.
13. Escalation: none.
14. Evidence: matching record ID if any.

**S8 — Synthesize**
1. Objective: one audio file.
2. Capability: `nova.voice.tts.synthesize.default`.
3. Input mapping: approved text, `voice_id`, model, format, language.
4. Dependencies: S7.
5. Preconditions: within caps; voice still valid.
6. Action / side effects: paid call; characters consumed.
7. Autonomy: A2 (covered by S6).
8. Timeout / retry: 120 s / 1 retry on a transient error after a history check.
9. Idempotency: request hash plus attempt number.
10. Postconditions: audio bytes.
11. Verification: bytes decode as the requested format.
12. Failure routes: `quota_exhausted` → `blocked`; timeout → history check, then one retry; other errors → `failed`.
13. Escalation: billing errors → tell the user.
14. Evidence: provider request ID, model, characters billed if reported.

**S9 — Validate the audio**
1. Objective: the audio is plausible for the script.
2. Capability: local audio inspection.
3. Input mapping: bytes, character count.
4. Dependencies: S8.
5. Preconditions: bytes present.
6. Action: decode; duration; speaking rate = characters ÷ duration within 6–30 characters per second; not silent; compute `sha256`.
7. Autonomy: A2.
8. Timeout / retry: 10 s / none.
9. Idempotency: read-only.
10. Postconditions: `duration_s`, `content_hash`.
11. Verification: all checks pass.
12. Failure routes: silent or truncated → `failed` for this attempt; offer one regeneration within caps.
13. Escalation: none.
14. Evidence: duration, rate, hash.

**S10 — Store and record**
1. Objective: durable file plus one audio record and one usage row.
2. Capability: `nova.content.assets.upload`; `nova.content.audio_records.add`.
3. Input mapping: bytes, `voiceovers/<audio_record_id>.<ext>`, record fields, characters, cost, `meta.job = content:voiceover`.
4. Dependencies: S9.
5. Preconditions: audio valid.
6. Action / side effects: upload; one transaction inserts the audio record and its usage row.
7. Autonomy: A2.
8. Timeout / retry: 30 s / 2 retries.
9. Idempotency: unique (`tenant_id`, request hash, `run_id`); hash path for the file.
10. Postconditions: record and usage row exist.
11. Verification: read back both; usage `units.characters` = character count.
12. Failure routes: upload fails → retry → `partially_completed` (paid, not stored).
13. Escalation: none.
14. Evidence: `audio_record_id`, `usage_id`, `audio_ref`.

**S11 — Independent check**
1. Objective: the audio says the script, and the quota moved as expected.
2. Capability: `nova.speech.transcribe` (independent verifier); `nova.voice.account.get_quota`.
3. Input mapping: `audio_ref`, script.
4. Dependencies: S10.
5. Preconditions: audio stored.
6. Action: transcribe back and compute word error rate; re-read quota and compare the delta with the characters sent.
7. Autonomy: A2.
8. Timeout / retry: 60 s / 1 retry.
9. Idempotency: read-only.
10. Postconditions: WER and quota delta recorded.
11. Verification: WER ≤ 10 % (`TBD` per language); quota delta within ± 5 % of characters.
12. Failure routes: WER high → flag to the user; verifier unavailable → mark `not_run` and disclose.
13. Escalation: none.
14. Evidence: WER, quota before and after.

**S12 — Deliver**
1. Objective: tell the user what was produced.
2. Capability: `nova.notify.owner.send`.
3. Input mapping: `voiceover_result`.
4. Dependencies: S11.
5. Preconditions: result assembled.
6. Action: send the playable reference, voice, duration, characters, cost and any flags.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 2 retries.
9. Idempotency: message key = `run_id`.
10. Postconditions: delivered.
11. Verification: delivery receipt.
12. Failure routes: delivery fails → result stays in the run record; `partially_completed`.
13. Escalation: none.
14. Evidence: message ID.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Input validity | request matches `request@0.1.0` | validator | valid | ask once, then `blocked` |
| Script approval | spoken text hash = approved script hash | S2 vs S8 input | equal | stop |
| Readiness | `ready` for synthesis and quota sufficient | S3 snapshot | pass | `blocked` with reason |
| Consent | cloned voice has a valid owner-granted consent covering the purpose | consent record | valid | `blocked`; disable voice if revoked |
| Clone approval | approval references the consent ID and sample hashes | approval record | equal | stop before sending samples |
| Audio plausibility | rate 6–30 chars/s, not silent | S9 | pass | regenerate once within caps |
| Single cost row | exactly one usage row per audio record | ledger | 1:1 | report the gap |
| Fidelity | WER ≤ threshold | S11 | pass | flag; never hide |
| Output schema | `voiceover_result` valid, `ai_generated = true` | validator | valid | fix fields; never fake values |

A provider returning audio proves only that audio exists, not that it speaks the approved script.

## J. Self-correction and recovery

- **Failure classes:** validation, unapproved script, provider not ready, quota exhausted, consent invalid, timeout, transient provider error, implausible audio, storage failure, verifier unavailable.
- **Permitted corrections:** ask once for missing input; use the allowlist variant when the live voice list lacks permission (stock voices only); one synthesis retry after a history check; one regeneration after a failed plausibility check; bounded retries of reads and writes.
- **Forbidden corrections:** changing script words; swapping to another voice or model without the user; using a cloned voice without a valid consent; recording zero cost; ignoring a high WER.
- **Maximum correction cycles:** 1 per step; 3 per run.
- **Reverification required after correction:** yes — S9 after regeneration; S4 consent check before any synthesis with a cloned voice.
- **Stop conditions:** caps reached; consent invalid; hard constraint fails.
- **Recovery or compensation:** disable a cloned voice whose consent lapsed; leave unlinked audio for retention cleanup.
- **What must be reported:** characters and cost per generation, any reuse, consent decisions, flags from the verifier.

## K. Independent challenge

- **Assumptions to challenge:** the voice is who the user thinks it is; the consent actually covers this purpose; the audio pronounces names and numbers correctly.
- **Alternative approach:** would the user's own recording be better for a personal announcement? Mention it when a cloned voice is requested for a high-stakes message.
- **Contradictory evidence:** transcript words that differ from the script; a quota delta far larger than the characters sent.
- **Could mislead:** AI speech presented as the owner speaking live; a consent record whose speaker differs from the sample speaker.
- **Separate verifier:** transcribe-back with a speech-to-text model independent of the TTS provider; consent validity is re-checked by the consent store, not by the planner's memory.
- **Findings requiring disclosure:** high WER, unverified quota, allowlist use, reuse of an earlier file.

## L. Human escalation

- **Triggers:** script not approved; over caps; quota exhausted; `payment_required` or `invalid_credential`; consent missing, expired or revoked; a clone request; doubt about the sample speaker.
- **Why the agent cannot decide:** money, consent and identity are human decisions.
- **Information presented:** script excerpt and hash, voice, characters, estimate, quota and reset time, consent summary (owner display name, scope, dates).
- **Decision requested:** approve the script or spend; choose a voice; approve a clone; renew billing (operator).
- **Options and consequences:** for example, waiting for the quota reset delays the audio; a stock voice avoids consent requirements.
- **Safe behaviour while waiting:** nothing generated or cloned.
- **If no response:** after 30 minutes the run ends `cancelled`.
- **Resume:** S3 readiness and S4 consent checks re-run before synthesis.

## M. Completion conditions

Mandatory: `voiceover_result` valid; script approved and spoken unchanged; voice valid (consent checked for cloned voices); audio validated and stored; exactly one audio record and one usage row; limitations disclosed; evidence references present.

- `completed` — audio generated (or reused with the user's yes), validated, stored, recorded and delivered.
- `partially_completed` — audio generated and paid but storing, recording or delivery failed, or the verifier flagged the audio.
- `blocked` — invalid input, provider not ready, quota exhausted or unreadable, consent invalid, or caps exceeded without approval.
- `failed` — synthesis or cloning failed, or the audio failed validation after the permitted regeneration.
- `cancelled` — the user declined the script, spend or clone, or approval timed out.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome; `voiceover_result`; step statuses S1–S12; verification results and failed checks; evidence references (script hash, approval IDs, consent ID, provider request ID, `audio_record_id`, `usage_id`); approvals and scope; side effects (characters, charges, clone created) and reconciliation status; corrections; unresolved challenges; limitations; escalation state; learning candidate. Never include provider keys, voice samples or full consent statements.

## O. Learning and revision

- **Outcome metrics:** first-attempt success rate, WER distribution, characters per run, cost per minute of audio, reuse rate, consent-block rate.
- **Evidence retained:** audio record, usage row, approvals, consent check results.
- **Data minimisation:** metrics stored without script text; voice samples never stored by this recipe beyond the provider transfer.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}`.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review; consent rules can never be relaxed by an automated revision.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (stock voice, approved test script) | audio stored, one record, one usage row, delivered | UNVERIFIED on Skill Port; observed in the reference implementation |
| Missing input (no voice) | asks once, then `blocked` | TBD |
| Invalid input (script over the provider limit) | asks to split; never truncates | TBD |
| Missing capability (TTS binding absent) | `blocked` with a dependency message | TBD |
| Missing permission (voice listing) | allowlist variant used for stock voices, disclosed | TBD |
| Quota exhausted | `blocked` with reset time | TBD |
| Permission denied (`voice:tts:create` missing) | `blocked`, no call | TBD |
| Unauthorized scope (another tenant's script or voice) | denied | TBD |
| Tenant isolation | cannot list another tenant's cloned voices or audio | TBD |
| Consent missing / revoked / purpose mismatch | `blocked`; revoked voice disabled | TBD |
| Clone without approval | nothing sent to the provider | TBD |
| Timeout on synthesis | history checked before one retry; no double charge | TBD |
| Duplicate / replay (same request hash) | reuse offered; unique key prevents a second record | TBD |
| False success (provider 200 with silent audio) | caught at S9; not reported as completed | TBD |
| Failed verification (WER above threshold) | flagged; `partially_completed` | TBD |
| Correction budget exhausted | stops safely | TBD |
| Escalation (`payment_required`) | user told; nothing generated | TBD |
| Cancellation (user declines the script) | `cancelled` | TBD |
| Compensation (consent revoked after creation) | voice disabled; no further use | TBD |
| Output schema and status | valid `voiceover_result`, correct status | TBD |
| Compatibility / regression (new provider model) | format and voice mapping still valid | TBD |

All tests use a sandbox tenant, test scripts and stock voices; consent tests use fictional consent records and synthetic samples created by the tester for that purpose. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, schema registry and approval UI — TBD (owner: publisher, with Skill Port).
- Per-run character cap, cost cap and price table — TBD (owner: operator).
- How the provider counts billable characters (spaces, markup) — TBD per provider.
- The consent flow that writes consent records is a dependency outside this recipe (owner: publisher).
- Legal requirements for voice cloning and AI-speech labelling vary by jurisdiction; the runtime must enforce them.
- WER thresholds per language — TBD.
- Retention for audio and consent statements — TBD (owner: operator, per local law).
