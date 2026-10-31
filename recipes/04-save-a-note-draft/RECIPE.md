# Save a note draft

## A. Identity and ownership

- **Recipe ID:** `nova.memory.save-a-note-draft` (proposed until assigned by Skill Port)
- **Name:** Save a note draft
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** writing
- **Tags:** notes, drafting, structuring, tags, human-confirmation · Vedaxi task type: Save a note draft
- **Summary:** The agent turns the user's rough text (typed fragments, dictation, pasted text) into a structured note — title, body and tags — without adding facts, saves it in the user's notes store with status `draft`, shows it to the user, and marks it `confirmed` only on the user's yes.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** An LLM for structuring and an independent fidelity check; a per-user notes store that supports create, read, update and a status field; an owner notification channel with reply buttons or text replies. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — turning rough text into a structured draft and saving it for the user to confirm runs in Nova's reference implementation. The notes-store binding is not confirmed for Skill Port, and nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.notes.draft` | Capability | Turn rough text into a faithful structured note | low | UNVERIFIED |
| `nova.notes.draft.screen` | Operation (read; rules + LLM) | Find secrets and sensitive items in the raw text | medium | UNVERIFIED |
| `nova.notes.draft.structure` | Operation (read, LLM) | Produce title, body and tags (and action items only if stated) | low | UNVERIFIED |
| `nova.notes.draft.check_fidelity` | Operation (read; rules + LLM) | Confirm nothing was added and nothing important dropped | low | UNVERIFIED |
| `nova.notes.store` | Capability | The user's notes store | medium | UNVERIFIED |
| `nova.notes.store.find_similar` | Operation (read) | Find existing notes with a similar title (duplicate hint only) | low | UNVERIFIED |
| `nova.notes.store.create_draft` | Operation (create) | Save one note with status `draft` | low | UNVERIFIED |
| `nova.notes.store.get` | Operation (read) | Read one note back (read-after-write) | low | UNVERIFIED |
| `nova.notes.store.update` | Operation (update) | Edit a draft's fields, or move status `draft` → `confirmed` or `discarded` | medium | UNVERIFIED |
| `nova.notify.owner` | Capability | Deliver a message to the requesting user's own channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Show the draft and ask Confirm / Edit / Discard | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.notes.draft.structure.default` — only the default variant is supported. The `source_kind` input (`typed`, `dictated`, `pasted`) changes cleanup rules inside the same contract (for example removing filler words from dictation); it is not a separate variant because the schema, permissions and side effects are identical.
- `nova.notes.draft.screen.default`, `nova.notes.draft.check_fidelity.default`, `nova.notes.store.find_similar.default`, `nova.notes.store.create_draft.default`, `nova.notes.store.get.default`, `nova.notes.store.update.default`, `nova.notify.owner.send.default` — only the default variant is supported.

**Variant selection policy:** single variants throughout; no fallback. If the notes store is unavailable, the draft is shown to the user unsaved and the run is reported as such — the agent never saves to another location (for example persistent memory) instead.

**Bindings:** `binding://nova/notes/screen`, `binding://nova/notes/structure`, `binding://nova/notes/fidelity`, `binding://nova/notes/find-similar`, `binding://nova/notes/create-draft`, `binding://nova/notes/get`, `binding://nova/notes/update`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: the assistant structures the note with its own LLM and saves it to the user's notes storage as a draft, then asks the user to confirm in chat.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Note | `schema://nova/save-a-note-draft/note@0.1.0` | the requesting user | create (draft), read, update (own draft only) |
| Screen result | `schema://nova/save-a-note-draft/screen-result@0.1.0` | this run only | create (in-run) |
| Fidelity report | `schema://nova/save-a-note-draft/fidelity-report@0.1.0` | this run only | create (in-run) |

**Authorization requirements (declared, never a grant):** `notes:read`, `notes:write` (create drafts), `notes:update` (own drafts created in this run, and only the fields and status transitions in Section F), `notify:owner`. LLM key `secret_ref://llm`; notes-store credential `secret_ref://notes-store` where the store needs one.

**Policy / trust:** risk class `low` — the note stays in the user's own store and nothing reaches a third party. Requires the secret screen (S2), the fidelity check (S4) and user confirmation before status `confirmed`.

## C. Intent and applicability

- **User problem:** The user has rough thoughts — a voice ramble, scattered bullets, pasted text — and wants a clean, findable note without rewriting it themselves.
- **Desired outcome:** A tidy draft that says only what the user said, saved where they can find it, waiting for their confirmation.
- **Trigger or entry conditions:** "make a note of this", "turn this into a note", "save this as a draft note", or rough text sent with a note request.
- **Applicable situations:** Meeting jottings, ideas, to-do brain dumps, call notes the user dictates, pasted text the user wants kept.
- **Out-of-scope situations:** Short facts to remember across conversations (use remember-a-note); money records (use the ledger or finance recipe); sending the note to anyone; editing or deleting existing confirmed notes; transcribing recordings of other people (needs a consent-aware recording recipe); generating new content the user did not provide ("write me a note about X").
- **Required inputs:** `raw_text` (see Section D).
- **Prerequisites:** A notes store bound to the user; an LLM.
- **Required capabilities:** `nova.notes.draft`, `nova.notes.store`, `nova.notify.owner` (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port; the notes-store binding must be resolved before activation.
- **Assumptions:** Verified in the reference implementation: structuring rough text into a draft and asking for confirmation. Assumed: the target store supports a `status` field or an equivalent draft flag.
- **Ambiguity policy:** Use a documented safe default — keep unclear phrases as written and mark them `[unclear]` rather than guessing; ask only when the raw text is empty or not a note request.
- **Vedaxi task type:** Save a note draft.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `raw_text` | string, 1–20,000 chars | yes | user; may be personal or confidential | — | Missing: ask for the text. Over 20,000 chars: ask to split, or save the first part and say so. |
| `title_hint` | string, ≤ 80 chars | no | user | generated | Over 80 chars: shorten and disclose. |
| `tags_hint` | array of strings, ≤ 5 | no | user | generated (1–5) | Invalid format: normalise to lowercase kebab-case and disclose. |
| `source_kind` | enum `typed`, `dictated`, `pasted` | no | runtime or inferred | `typed` | — |
| `language` | BCP 47 tag | no | inferred from the text | the text's language | The note stays in the language of the raw text; no translation unless asked. |

```json
{"$id":"schema://nova/save-a-note-draft/request@0.1.0","type":"object","required":["raw_text"],
 "properties":{"raw_text":{"type":"string","minLength":1,"maxLength":20000},"title_hint":{"type":"string","maxLength":80},
 "tags_hint":{"type":"array","maxItems":5,"items":{"type":"string","maxLength":30}},
 "source_kind":{"enum":["typed","dictated","pasted"]},"language":{"type":"string"}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `note` | object `schema://nova/save-a-note-draft/note@0.1.0` | `note_id`, `status`, `title`, `body`, `tags`, `source_sha256` | the user's notes store; preview to the user's channel | the user's own content; owner only | read-after-write and fidelity report (Section I) |

```json
{"$id":"schema://nova/save-a-note-draft/note@0.1.0","type":"object",
 "required":["note_id","status","title","body","tags","source_sha256"],
 "properties":{"note_id":{"type":"string"},"status":{"enum":["draft","confirmed","discarded"]},
 "title":{"type":"string","minLength":1,"maxLength":80},"body":{"type":"string","minLength":1,"maxLength":20000},
 "tags":{"type":"array","minItems":1,"maxItems":5,"items":{"type":"string","pattern":"^[a-z0-9]+(-[a-z0-9]+)*$","maxLength":30}},
 "action_items":{"type":"array","items":{"type":"string"}},"language":{"type":"string"},
 "source_kind":{"enum":["typed","dictated","pasted"]},"source_sha256":{"type":"string"},"content_sha256":{"type":"string"},
 "created_at":{"type":"string","format":"date-time"},"updated_at":{"type":"string","format":"date-time"}}}
```

Helper schemas (draft, version 0.1.0; each `$id` matches the manifest reference):

```json
{"$id":"schema://nova/save-a-note-draft/screen-result@0.1.0","type":"object","required":["findings","action"],"properties":{"findings":{"type":"array","items":{"type":"object","required":["kind","span_start","span_end"],"properties":{"kind":{"enum":["secret","payment_data","government_id","special_category"]},"span_start":{"type":"integer","minimum":0},"span_end":{"type":"integer","minimum":0}}}},"action":{"enum":["none","redact_proposed"]},"user_choice":{"enum":["redact","keep","cancel",null]}}}
{"$id":"schema://nova/save-a-note-draft/fidelity-report@0.1.0","type":"object","required":["added_facts","dropped_points","unclear_marked","passed"],"properties":{"added_facts":{"type":"array","items":{"type":"string"}},"dropped_points":{"type":"array","items":{"type":"object","required":["text","reason"],"properties":{"text":{"type":"string"},"reason":{"enum":["filler","duplicate","user_request"]}}}},"unclear_marked":{"type":"integer","minimum":0},"passed":{"type":"boolean"}}}
{"$id":"schema://nova/save-a-note-draft/note-list@0.1.0","type":"object","required":["notes"],"properties":{"notes":{"type":"array","maxItems":10,"items":{"type":"object","required":["note_id","title","status"],"properties":{"note_id":{"type":"string"},"title":{"type":"string"},"tags":{"type":"array","items":{"type":"string"}},"status":{"enum":["draft","confirmed","discarded"]},"updated_at":{"type":"string","format":"date-time"}}}}}}
{"$id":"schema://nova/save-a-note-draft/note-ref@0.1.0","type":"object","anyOf":[{"required":["note_id"]},{"required":["idempotency_key"]}],"properties":{"note_id":{"type":"string"},"idempotency_key":{"type":"string"}}}
{"$id":"schema://nova/save-a-note-draft/note-update@0.1.0","type":"object","required":["note_id","expected_status","expected_content_sha256"],"properties":{"note_id":{"type":"string"},"expected_status":{"const":"draft"},"expected_content_sha256":{"type":"string"},"changes":{"type":"object","properties":{"title":{"type":"string","minLength":1,"maxLength":80},"body":{"type":"string","minLength":1,"maxLength":20000},"tags":{"type":"array","minItems":1,"maxItems":5,"items":{"type":"string","pattern":"^[a-z0-9]+(-[a-z0-9]+)*$"}},"action_items":{"type":"array","items":{"type":"string"}}}},"new_status":{"enum":["draft","confirmed","discarded"]},"approval_id":{"type":["string","null"]}}}
{"$id":"schema://nova/save-a-note-draft/run-result@0.1.0","type":"object","required":["status","note_status","fidelity"],"properties":{"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},"note_id":{"type":["string","null"]},"note_status":{"enum":["draft","confirmed","discarded","unsaved"]},"fidelity":{"$ref":"schema://nova/save-a-note-draft/fidelity-report@0.1.0"},"similar_note_ids":{"type":"array","items":{"type":"string"}}}}
{"$id":"schema://nova/save-a-note-draft/delivery@0.1.0","type":"object","required":["message_id","status"],"properties":{"message_id":{"type":"string"},"status":{"enum":["delivered","failed"]},"delivered_at":{"type":["string","null"],"format":"date-time"},"channel":{"type":"string"}}}
```

**Example (fictional)**

Input: `{"raw_text":"ok so call w/ jane doe re the q3 offsite — she says venue A is booked, venue B maybe, need to check catering by fri, um also budget cap is 5k, I'll send her the agenda","source_kind":"dictated"}`

Output: `{"note_id":"note_0311","status":"draft","title":"Q3 offsite — call with Jane Doe","body":"- Venue A is booked.\n- Venue B is a maybe.\n- Check catering by Friday.\n- Budget cap: 5k.\n- I will send Jane the agenda.","tags":["q3-offsite","venues","call-notes"],"action_items":["Check catering by Friday","Send Jane the agenda"],"language":"en","source_kind":"dictated","source_sha256":"<sha256 of raw text>","created_at":"2026-10-31T10:02:00Z","updated_at":"2026-10-31T10:02:00Z"}` — fidelity report: `{"added_facts":[],"dropped_points":[{"text":"ok so","reason":"filler"},{"text":"um also","reason":"filler"}],"unclear_marked":0,"passed":true}`. Note that "5k" keeps its original form: no currency is added because the user did not state one.

## E. Constraints and budgets

- **Hard constraints:** add no facts — no names, numbers, currencies, dates, owners or commitments that are not in the raw text; keep every substantive point (anything dropped is listed with a reason); action items only when the text states them; save only with status `draft`; move to `confirmed` only on the user's explicit yes for this note; never send, share or publish the note; never store secrets (passwords, PINs, one-time codes, API keys, full card or bank numbers) — propose redaction and save only after the user chooses; the user's own notes store only.
- **Soft constraints:** title ≤ 80 chars and specific; bullets for lists, short paragraphs otherwise; 1–5 tags in lowercase kebab-case; keep the user's language and wording for key terms; relative dates kept as written.
- **Resource and tenant scope:** the requesting user's notes store only.
- **Data sources allowed:** the raw text; the user's existing note titles (for the duplicate hint only).
- **External transfers permitted:** the raw text to the configured LLM for structuring and the fidelity check only.
- **Time limit:** structuring and save ≤ 90 s; confirmation wait up to 24 hours (TBD), after which the draft simply stays a draft.
- **Cost or token budget:** `TBD` (set by the operator); scales with raw-text length.
- **Maximum steps:** 9 (S1–S9).
- **Maximum retries and correction cycles:** structuring: 1 regeneration; store writes: 2 retries after a read-by-key; edit rounds requested by the user: up to 3 per run; correction cycles: 1 per step, 3 per run.
- **Rate limits and concurrency:** one draft per request; store limits as configured.
- **Retention and logging constraints:** the note persists in the user's store until the user deletes it (outside this recipe); discarded drafts are kept with status `discarded` for 30 days (TBD) then purged by the store; run logs record IDs and hashes, never note text.
- **Prohibited actions:** hard-deleting any note; editing confirmed notes or notes not created in this run; saving to persistent memory instead of the notes store; translating or summarising away content without being asked.
- **Stop conditions:** empty input; the user declines a redaction choice; store unavailable after retries; the user discards or cancels.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Note with status `draft` (one per request) | S2 screen resolved; S4 fidelity passed | Allowed (the request is the approval for a draft) | read-after-write matches title, body hash and tags |
| Read | Own notes (titles for duplicate hint; the new draft) | own store | Allowed | — |
| Update | The new draft's `title`, `body`, `tags`, `action_items` | status `draft`; user asked for the edit | Allowed (the edit request is the approval) | read-after-write |
| Update | The new draft's `status` → `confirmed` | user said yes to this exact version | Approval Required | status `confirmed` on re-read; content hash unchanged |
| Update | The new draft's `status` → `discarded` | user chose Discard | Approval Required (the Discard choice) | status `discarded` on re-read |
| Delete | any | — | Not Allowed (discard is a status change) | — |
| External side effect | preview and prompt to the requesting user | draft saved (or unsaved fallback) | Allowed | delivery receipt |

- **Idempotency:** create key = hash(`user_id`, `run_id`, `source_sha256`). A repeated create with the same key returns the existing `note_id`. Status changes are conditional on the current status (`draft`) and the content hash the user approved.
- **Expected side effects:** one note record in the user's store.
- **Compensation:** Discard sets status `discarded`; a confirmed note can be set back to `draft` only by a new, explicit user request (outside this recipe).
- **Irreversible:** none within this recipe (no sends, no hard deletes).
- **Ambiguous outcomes:** if a create times out, `get` by idempotency key before retrying; if a status update times out, re-read status before retrying.

## G. Autonomy and approval policy

- **Default autonomy level:** A1 — supervised execution.
- **Planning level:** A2 (screen, structure, check).
- **Execution level:** A1 — saving a draft is covered by the request; confirming needs the user's yes.
- **Verification level:** A2, with an independent fidelity pass.
- **Recovery level:** A1.
- **Approval required for:** status `confirmed`; status `discarded`; saving text that contains a flagged secret (the user chooses redact or keep — keep is refused for secrets, allowed for other flagged sensitive items with a warning).
- **Actions never permitted:** adding facts; sending or sharing the note; hard deletion; editing other notes; storing secrets.
- **Conditions requiring clarification:** empty or non-note input; a redaction choice.
- **Conditions requiring escalation:** none beyond the confirmation and redaction prompts.
- **Safe state after stop, cancellation or escalation:** at most one note with status `draft` or `discarded`; nothing confirmed without a yes; nothing sent.

Approval to confirm covers the exact version shown (content hash). Any later edit returns the note to needing confirmation.

## H. Execution plan

**S1 — Validate the request**
1. Objective: usable raw text and a note request.
2. Capability: none (local validation).
3. Input mapping: user message (and any attached text) → `request@0.1.0`.
4. Dependencies: none.
5. Preconditions: a note request.
6. Action / side effects: validate; compute `source_sha256`; no side effects.
7. Autonomy: A2; no approval.
8. Timeout / retry: 5 s / none.
9. Idempotency: not applicable.
10. Postconditions: schema-valid request and source hash.
11. Verification: JSON Schema validation passes.
12. Failure routes: empty text → ask once → else `blocked`; over the limit → ask to split.
13. Escalation triggers: none.
14. Evidence: source hash and length.

**S2 — Screen for secrets and sensitive items**
1. Objective: keep secrets out of the store.
2. Capability: `nova.notes.draft.screen`.
3. Input mapping: `raw_text`.
4. Dependencies: S1.
5. Preconditions: request valid.
6. Action / side effects: pattern rules (card and account formats with checksum where applicable, "password", "PIN", one-time codes, key-like strings) plus LLM for special-category data; no side effects.
7. Autonomy: A2; A0 for the redaction choice.
8. Timeout / retry: 15 s / 1 retry.
9. Idempotency: not applicable.
10. Postconditions: `screen-result@0.1.0`; secrets replaced by `[redacted]` in the working text after the user's choice.
11. Verification: rule matches cannot be overridden by the LLM; no secret pattern remains in the working text.
12. Failure routes: secrets found → ask "Redact and continue, or cancel?" → cancel → `cancelled`; other sensitive items → warn and continue unless the user objects.
13. Escalation triggers: the user insists on keeping a secret → refuse that part; offer to continue redacted.
14. Evidence: finding kinds and counts (never the values).

**S3 — Structure the draft**
1. Objective: title, body, tags and stated action items.
2. Capability: `nova.notes.draft.structure`.
3. Input mapping: working text, `title_hint`, `tags_hint`, `source_kind`, `language`.
4. Dependencies: S2.
5. Preconditions: screen resolved.
6. Action / side effects: remove filler (for `dictated`), group related points, keep wording for key terms, mark unclear phrases `[unclear]`; no side effects.
7. Autonomy: A2.
8. Timeout / retry: 60 s / 1 retry.
9. Idempotency: not applicable.
10. Postconditions: a candidate `note@0.1.0` without `note_id`.
11. Verification: schema constraints (title length, tag pattern, 1–5 tags).
12. Failure routes: schema errors → regenerate once → else `failed`.
13. Escalation triggers: none.
14. Evidence: candidate draft.

**S4 — Check fidelity independently**
1. Objective: prove nothing was added and nothing substantive dropped.
2. Capability: `nova.notes.draft.check_fidelity` (rules plus a separate LLM pass that sees only the working text and the candidate).
3. Input mapping: working text, candidate draft.
4. Dependencies: S3.
5. Preconditions: candidate exists.
6. Action / side effects: every name, number, date and currency in the candidate must appear in the working text; every substantive point in the working text must appear in the body or in `dropped_points` with reason `filler` or `duplicate`; action items must be stated in the text; no side effects.
7. Autonomy: A2.
8. Timeout / retry: 30 s / 1 retry.
9. Idempotency: not applicable.
10. Postconditions: `fidelity-report@0.1.0` with `passed`.
11. Verification: `added_facts` is empty; no dropped point has a reason other than `filler`, `duplicate` or `user_request`.
12. Failure routes: fails → regenerate S3 once with the report as feedback → still fails → remove the offending additions mechanically and re-check → still fails → `failed`.
13. Escalation triggers: none.
14. Evidence: fidelity report.

**S5 — Duplicate hint**
1. Objective: tell the user if a similar note already exists.
2. Capability: `nova.notes.store.find_similar`.
3. Input mapping: candidate title and tags.
4. Dependencies: S4.
5. Preconditions: fidelity passed.
6. Action / side effects: read-only.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 1 retry.
9. Idempotency: read-only.
10. Postconditions: zero or more similar note IDs.
11. Verification: results are the user's own notes.
12. Failure routes: unavailable → skip and continue (the hint is optional).
13. Escalation triggers: none. Similar notes are never merged automatically.
14. Evidence: similar note IDs.

**S6 — Save as draft**
1. Objective: persist the note with status `draft`.
2. Capability: `nova.notes.store.create_draft`.
3. Input mapping: candidate draft, idempotency key.
4. Dependencies: S5.
5. Preconditions: fidelity passed; screen resolved.
6. Action / side effects: one note record created.
7. Autonomy: A1, covered by the request.
8. Timeout / retry: 15 s / 2 retries, each preceded by `get` by key.
9. Idempotency: key = hash(`user_id`, `run_id`, `source_sha256`).
10. Postconditions: `note_id` assigned; status `draft`.
11. Verification: see S7.
12. Failure routes: store unavailable after retries → show the draft unsaved in S8 → `partially_completed`.
13. Escalation triggers: none.
14. Evidence: `note_id`.

**S7 — Read-after-write**
1. Objective: prove the stored note equals the checked draft.
2. Capability: `nova.notes.store.get`.
3. Input mapping: `note_id`.
4. Dependencies: S6.
5. Preconditions: create reported success.
6. Action / side effects: read-only.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: stored title, body hash, tags and status match.
11. Verification: exact match; status `draft`.
12. Failure routes: mismatch → one corrective `update` → re-read → still wrong → `failed`, reporting the stored state.
13. Escalation triggers: none.
14. Evidence: stored content hash.

**S8 — Present for confirmation**
1. Objective: let the user confirm, edit or discard.
2. Capability: `nova.notify.owner.send`.
3. Input mapping: title, body, tags, action items, fidelity summary ("2 filler phrases removed; nothing added"), similar notes, note status.
4. Dependencies: S7 (or S6 on the unsaved route).
5. Preconditions: draft verified or shown as unsaved.
6. Action / side effects: one message with Confirm / Edit / Discard.
7. Autonomy: A0 — human decision.
8. Timeout / retry: send 15 s / 2 retries; wait up to 24 hours (TBD).
9. Idempotency: message key = `run_id`.
10. Postconditions: a decision, or a timeout with the note left as `draft`.
11. Verification: delivery receipt; the preview hash equals the stored hash.
12. Failure routes: delivery fails → note stays `draft` → `partially_completed`.
13. Escalation triggers: none.
14. Evidence: message ID, decision and timestamp.

**S9 — Apply the decision and report**
1. Objective: carry out exactly the user's choice.
2. Capability: `nova.notes.store.update`, `nova.notes.store.get`, `nova.notify.owner.send`.
3. Input mapping: decision, `note_id`, approved content hash, edit instructions.
4. Dependencies: S8.
5. Preconditions: status `draft`; for Confirm, the stored hash equals the hash shown.
6. Action / side effects: Confirm → status `confirmed`; Edit → apply the edit (re-run S4 on the new content), save, and return to S8 (at most 3 rounds); Discard → status `discarded`.
7. Autonomy: A1 under the user's decision.
8. Timeout / retry: 15 s / 2 retries, each preceded by a status read.
9. Idempotency: conditional update on status `draft` and the approved hash.
10. Postconditions: final status matches the decision.
11. Verification: re-read status and hash.
12. Failure routes: conditional update rejected (the note changed) → re-read and present again; edit rounds exhausted → leave as `draft` and say so.
13. Escalation triggers: none.
14. Evidence: final status, hash, message ID.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Input validity | request matches `request@0.1.0` | validator | valid | ask once, then `blocked` |
| No secrets stored | no secret pattern in stored text | S2 rules on working text and stored note | none | redact or cancel |
| No added facts | every name, number, date, currency in the note is in the source | S4 | `added_facts` empty | regenerate or remove |
| Nothing lost | every substantive point kept or listed as filler / duplicate | S4 | passed | regenerate |
| Draft status | new note saved as `draft` | S7 read | `draft` | correct once, then `failed` |
| Write truth | stored content equals the checked draft | S7 | hashes equal | correct once, then `failed` |
| Confirmation scope | `confirmed` only with a yes for the shown hash | S9 | hash equal | do not confirm |
| Scope | only the user's store and this run's note touched | store audit | own scope | stop and report |
| Output schema | `note` and `run-result` valid | validator | valid | fix fields; never fake values |

A store returning success proves only that a write happened; S7 proves what was written.

## J. Self-correction and recovery

- **Failure classes:** validation, secret found, schema error, fidelity failure, store unavailable, timeout, read-after-write mismatch, conditional-update conflict, permission denial.
- **Permitted corrections:** ask once; redact with the user's choice; regenerate once with fidelity feedback; mechanically strip added facts; retry writes after reading by key; one corrective update after a mismatch; re-present after a conflict.
- **Forbidden corrections:** keep a secret; confirm without a yes; mark `confirmed` after a failed fidelity check; save elsewhere when the store fails; hard-delete to "clean up".
- **Maximum correction cycles:** 1 per step; 3 per run (user-requested edit rounds are separate, up to 3).
- **Reverification required after correction:** yes — S4 after any content change; S7 after any write.
- **Stop conditions:** correction budget exhausted; hard constraint fails.
- **Recovery or compensation:** set a wrongly created draft to `discarded` and tell the user.
- **What must be reported:** redactions, filler removed, `[unclear]` marks, similar notes, final status.

## K. Independent challenge

- **Assumptions to challenge:** a grouping or heading does not change meaning; "5k" means the same without a currency; an implied task really is the user's action item.
- **Alternative approach:** if the text is a single short fact, suggest remember-a-note; if it is mostly money entries, suggest the ledger.
- **Contradictory evidence:** points in the raw text that contradict each other — keep both and flag them rather than resolving.
- **Could mislead:** a confident title that overstates a tentative point ("Venue B booked" when the user said "maybe"); tags that imply a project the user did not name.
- **Separate verifier:** S4 runs separately from S3 and sees only the source and candidate.
- **Findings requiring disclosure:** redactions, removed filler, `[unclear]` marks, contradictions kept, unsaved fallback.

## L. Human escalation

- **Triggers:** secret found; confirmation; edit rounds exhausted.
- **Why the agent cannot decide:** whether the draft says what the user meant, and whether to keep sensitive text, are the user's call.
- **Information presented:** the full draft, fidelity summary, redaction findings (kinds only), similar notes.
- **Decision requested:** Confirm / Edit / Discard; Redact / Cancel.
- **Options and consequences:** Confirm marks the note final; Discard hides it and it is purged later; no answer leaves it as a draft.
- **Safe behaviour while waiting:** the note stays `draft`; nothing sent.
- **If no response:** after the wait window the run ends `partially_completed` with the draft saved and listed as awaiting confirmation.
- **Resume:** a later "confirm that note" re-reads the note and checks its hash before confirming.

## M. Completion conditions

Mandatory: `note` valid; S4 passed; read-after-write matched; no secrets stored; status changes only on the user's decision; the user told the final state; evidence references present.

- `completed` — the draft was saved and verified, presented, and the user confirmed it (status `confirmed`) or explicitly chose to keep it as a draft.
- `partially_completed` — the draft was saved and verified but the user has not responded (status `draft`, awaiting confirmation); or the store failed and the draft was shown unsaved; or edit rounds ran out.
- `blocked` — empty or invalid input after one question.
- `failed` — fidelity could not be satisfied, or the stored note could not be made to match.
- `cancelled` — the user cancelled at the redaction prompt or discarded the draft (status `discarded`).

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome; `note` (or its ID and status when the channel is shared); `run-result` with the fidelity report; step statuses S1–S9; verification results and failed checks; evidence references (source hash, `note_id`, stored hash, decision message ID); approvals and scope (confirm or discard with the approved hash); side effects (note created, status changes) and reconciliation status; corrections performed; unresolved challenges (contradictions kept); limitations; escalation state; learning candidate. Never include redacted values or credentials.

## O. Learning and revision

- **Outcome metrics:** confirm rate without edits, edit rounds per note, fidelity-failure rate, redaction rate, unanswered-draft rate, time to confirmation.
- **Evidence retained:** run record with hashes and IDs, per the retention policy.
- **Data minimisation:** metrics stored without note text.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}`.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review; no silent changes to trusted behaviour.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (dictated ramble) | faithful draft saved, verified, confirmed on yes | UNVERIFIED on Skill Port; observed in the reference implementation |
| Missing input | asks once, then `blocked` | TBD |
| Invalid input (25,000 chars) | asks to split; nothing saved silently | TBD |
| Secret in text (a PIN) | redaction prompt; stored text has `[redacted]` | TBD |
| Added-fact trap ("5k" without currency) | no currency added | TBD |
| Missing capability (notes store absent) | draft shown unsaved; `partially_completed` | TBD |
| Permission denied (`notes:write` missing) | nothing saved; `blocked` with reason | TBD |
| Unauthorized scope (update another user's note) | denied | TBD |
| Tenant isolation | `find_similar` returns only own notes | TBD |
| Timeout on create | `get` by key before retry; one note only | TBD |
| Retry on transient store error | bounded; then unsaved fallback | TBD |
| Duplicate / replay | same key returns the same `note_id` | TBD |
| False success (store OK, body truncated) | caught at S7; corrected or `failed` | TBD |
| Failed verification (draft drops a point) | regenerated; point restored | TBD |
| Correction budget exhausted | `failed`; nothing confirmed | TBD |
| Escalation (edit rounds exhausted) | left as `draft`; user told | TBD |
| Cancellation (Discard) | status `discarded`; no hard delete | TBD |
| Rollback / compensation (wrong draft created) | set to `discarded`; verified | TBD |
| Confirm-after-change (note edited elsewhere) | conditional update rejected; re-presented | TBD |
| Output schema and status | valid `note`, correct status | TBD |
| Compatibility / regression | passes on a new notes-store or LLM version | TBD |

All tests use test users and fictional text. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, schema registry and the notes-store binding — TBD (owner: publisher, with Skill Port).
- Confirmation wait window and discarded-draft purge period — TBD (owner: operator).
- Whether the target store supports conditional updates on a content hash; if not, a status re-read before every update is the minimum.
- Fidelity checks use rules plus an LLM; paraphrase can still shift tone or emphasis, which is why the user confirms.
- Very long inputs may exceed the LLM context; splitting is manual in this version.
