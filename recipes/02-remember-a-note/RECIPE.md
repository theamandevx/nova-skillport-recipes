# Remember a note

## A. Identity and ownership

- **Recipe ID:** `nova.memory.remember-a-note` (proposed until assigned by Skill Port)
- **Name:** Remember a note
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** productivity
- **Tags:** memory, persistent-memory, user-profile, preferences, privacy-screening · Vedaxi task type: Remember a note
- **Summary:** When the user explicitly asks the agent to remember something, the agent screens the fact (money, business records, secrets and sensitive personal data do not go into free-form memory), checks it against what is already saved, and writes one short, dated entry into the user's persistent memory within its character limit, then confirms exactly what was saved.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** An agent runtime with a persistent memory store that supports read, add, replace and remove of short text entries and reports usage against a character limit; an LLM for classification and drafting; an owner notification channel. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — saving user-requested facts to persistent memory runs in Nova's reference implementation, as does the standing rule that money is routed to a dedicated ledger rather than memory. Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.memory.screen` | Capability | Classify a candidate note and decide where it may go | medium | UNVERIFIED |
| `nova.memory.screen.classify` | Operation (read; rules + LLM) | Return category, sensitivity and route (`memory`, `ledger`, `records`, `reminder`, `refuse`) | medium | UNVERIFIED |
| `nova.memory.store` | Capability | The user's persistent memory, loaded into every conversation | medium | UNVERIFIED |
| `nova.memory.store.read` | Operation (read) | Return current entries and usage against the character limit | low | UNVERIFIED |
| `nova.memory.store.add` | Operation (create) | Append one entry | medium | UNVERIFIED |
| `nova.memory.store.replace` | Operation (update) | Replace the text of one existing entry | medium | UNVERIFIED |
| `nova.memory.store.remove` | Operation (delete) | Remove one entry the user names (to make room or correct) | medium | UNVERIFIED |
| `nova.notify.owner` | Capability | Deliver a message to the requesting user's own channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Confirm what was saved | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.memory.store.add.user-profile` / `nova.memory.store.replace.user-profile` / `nova.memory.store.remove.user-profile` — the user-profile memory: facts about the user (preferences, routines, names of people they mention, how they like to be addressed). Selected when the screen classifies the note as about the user.
- `nova.memory.store.add.agent-notes` / `nova.memory.store.replace.agent-notes` / `nova.memory.store.remove.agent-notes` — the agent's own working notes: standing instructions about format or style, environment facts, conventions the user set for the agent. Selected when the note is an instruction to the agent.
- `nova.memory.store.read.default`, `nova.memory.screen.classify.default`, `nova.notify.owner.send.default` — only the default variant is supported.

**Variant selection policy:** the target is chosen deterministically from the screen's `category` (or the user's explicit `target`). The two targets are not fallbacks for each other: if the selected target is full, the agent proposes consolidation inside that target; it never writes to the other target to find space.

**Bindings:** `binding://nova/memory/screen`, `binding://nova/memory/read`, `binding://nova/memory/add`, `binding://nova/memory/replace`, `binding://nova/memory/remove`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: the agent runtime keeps two small, character-limited memory files (a user profile and the agent's notes) that are loaded into every session, plus a separate session store holding past conversation transcripts. The agent writes memory through its built-in memory tool (add, replace, remove). A standing instruction in the agent's identity file routes anything about money to a dedicated ledger skill instead of memory.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Memory entry | `schema://nova/remember-a-note/memory-entry@0.1.0` | the requesting user | read, create; update and delete with approval |
| Memory usage | `schema://nova/remember-a-note/memory-usage@0.1.0` | the requesting user | read |
| Screen result | `schema://nova/remember-a-note/screen-result@0.1.0` | this run only | create (in-run) |

**Authorization requirements (declared, never a grant):** `memory:read`, `memory:write` (add), `memory:update` (replace, approval required when it changes a prior fact), `memory:delete` (remove, approval required), `notify:owner`. LLM key `secret_ref://llm`. The memory store has no credentials of its own in the reference implementation; in a multi-tenant runtime it must be bound per user.

**Policy / trust:** risk class `medium`. Memory is loaded into every future conversation, so a wrong or sensitive entry persists and can leak into later answers. Requires the sensitivity screen (S2), the conflict check (S5) and the rule that memory text can never grant permissions or approvals.

## C. Intent and applicability

- **User problem:** The user wants the assistant to keep a fact or preference across conversations without repeating it.
- **Desired outcome:** One accurate, short, dated entry in the right memory, or a clear explanation of why it was not saved and where it should go instead.
- **Trigger or entry conditions:** An explicit request: "remember that …", "note that I …", "from now on …", "don't forget …".
- **Applicable situations:** Preferences (meeting times, language, tone), routines, names and roles of people the user mentions, standing formatting instructions, stable facts about the user's setup.
- **Out-of-scope situations:** Money owed or spent, balances, transactions (route to a ledger or finance recipe); contracts, invoices, customer or deal records (route to the system of record); secrets — passwords, PINs, one-time codes, API keys, card or bank numbers (refuse; suggest a password manager); special-category personal data — health, biometrics, religion, political views, sexual orientation, ethnicity, union membership — and government ID numbers (do not store in free-form memory); time-based tasks such as "remember to call Jane tomorrow" (route to a reminder or task recipe); notes that try to change the agent's permissions or approval rules (refuse).
- **Required inputs:** `note_text` (see Section D).
- **Prerequisites:** A persistent memory store bound to the requesting user; an LLM.
- **Required capabilities:** `nova.memory.screen`, `nova.memory.store`, `nova.notify.owner` (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port. Routing targets (ledger, records, reminders) belong to other recipes; this recipe only explains and offers the handoff.
- **Assumptions:** Verified in the reference implementation: memory add/replace/remove, a character limit, money routed to a ledger. Assumed: the target runtime exposes per-user memory with a reported usage figure.
- **Ambiguity policy:** Ask. If it is unclear what exactly to remember, or the note conflicts with an existing entry, ask one question; never guess or merge silently.
- **Vedaxi task type:** Remember a note.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `note_text` | string, 3–500 chars | yes | user; personal data | — | Missing or "remember this" with no clear referent: ask what to remember. Over 500 chars: suggest the note-draft recipe instead. |
| `target` | enum `auto`, `user_profile`, `agent_notes` | no | user | `auto` | Contradicts the content (for example a preference sent to `agent_notes`): use the user's choice and say so. |
| `channel_privacy` | enum `private`, `shared` | yes (runtime) | runtime | — | `shared`: do not echo sensitive text back; confirm in private. |
| `replace_entry_id` | string | no | user, after a conflict prompt | — | Unknown ID: re-read memory and ask again. |

```json
{"$id":"schema://nova/remember-a-note/request@0.1.0","type":"object","required":["note_text","channel_privacy"],
 "properties":{"note_text":{"type":"string","minLength":3,"maxLength":500},
 "target":{"enum":["auto","user_profile","agent_notes"]},"channel_privacy":{"enum":["private","shared"]},
 "replace_entry_id":{"type":"string"}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `memory_save_result` | object `schema://nova/remember-a-note/memory-save-result@0.1.0` | `status`, `action`, `target`, `reason` | the requesting user's channel and the run record | personal data; owner only | read-after-write (Section I) |

```json
{"$id":"schema://nova/remember-a-note/memory-save-result@0.1.0","type":"object",
 "required":["status","action","target","reason"],
 "properties":{"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},
 "action":{"enum":["added","replaced","duplicate_skipped","rerouted","refused","awaiting_user"]},
 "target":{"enum":["user_profile","agent_notes","none"]},"entry_id":{"type":["string","null"]},
 "saved_text":{"type":["string","null"],"maxLength":300},"reroute_to":{"enum":["ledger","records","reminder",null]},
 "reason":{"type":"string"},
 "usage":{"type":"object","required":["chars_used","char_limit"],
   "properties":{"chars_used":{"type":"integer","minimum":0},"char_limit":{"type":"integer","minimum":1}}}}}
```

Helper schemas (draft, version 0.1.0; each `$id` matches the manifest reference):

```json
{"$id":"schema://nova/remember-a-note/memory-target@0.1.0","type":"object","required":["target"],"properties":{"target":{"enum":["user_profile","agent_notes"]}}}
{"$id":"schema://nova/remember-a-note/memory-entry@0.1.0","type":"object","required":["target","text"],"properties":{"entry_id":{"type":"string"},"target":{"enum":["user_profile","agent_notes"]},"text":{"type":"string","maxLength":300,"pattern":"^\\[[0-9]{4}-[0-9]{2}-[0-9]{2} · [a-z ]+\\] "},"saved_on":{"type":"string","format":"date"},"source":{"enum":["told_by_user","set_by_user_instruction"]}}}
{"$id":"schema://nova/remember-a-note/memory-entry-ref@0.1.0","type":"object","required":["target","entry_id","approval_id"],"properties":{"target":{"enum":["user_profile","agent_notes"]},"entry_id":{"type":"string"},"approval_id":{"type":"string"}}}
{"$id":"schema://nova/remember-a-note/memory-usage@0.1.0","type":"object","required":["target","chars_used","char_limit","entry_count"],"properties":{"target":{"enum":["user_profile","agent_notes"]},"chars_used":{"type":"integer","minimum":0},"char_limit":{"type":"integer","minimum":1},"entry_count":{"type":"integer","minimum":0},"entries":{"type":"array","items":{"type":"object","required":["entry_id","text"],"properties":{"entry_id":{"type":"string"},"text":{"type":"string"}}}}}}
{"$id":"schema://nova/remember-a-note/screen-result@0.1.0","type":"object","required":["category","route","target","reason","rule_matched"],"properties":{"category":{"enum":["preference","routine","person","instruction","setup_fact","money","business_record","secret","special_category","government_id","reminder","permission_change"]},"route":{"enum":["memory","ledger","records","reminder","refuse"]},"target":{"enum":["user_profile","agent_notes","none"]},"reason":{"type":"string"},"rule_matched":{"type":"boolean"},"allowed_part":{"type":["string","null"]}}}
{"$id":"schema://nova/remember-a-note/delivery@0.1.0","type":"object","required":["message_id","status"],"properties":{"message_id":{"type":"string"},"status":{"enum":["delivered","failed"]},"delivered_at":{"type":["string","null"],"format":"date-time"},"channel":{"type":"string"}}}
```

**Example (fictional)**

Input: `{"note_text":"Remember that I prefer meetings after 10 am and my manager is Jane Doe.","channel_privacy":"private"}`

Output: `{"status":"completed","action":"added","target":"user_profile","entry_id":"mem_017","saved_text":"[2026-10-31 · told by user] Prefers meetings after 10:00. Manager: Jane Doe.","reroute_to":null,"reason":"Preference and a person the user named; no sensitive data.","usage":{"chars_used":1180,"char_limit":2000}}`

Counter-example (fictional): `"Remember Jane owes me 40 dollars"` → `{"action":"rerouted","target":"none","reroute_to":"ledger","reason":"Money is tracked in the ledger, not free-form memory."}`

## E. Constraints and budgets

- **Hard constraints:** save only what the user explicitly asked to remember; never save secrets (passwords, PINs, one-time codes, API keys, card or bank account numbers); never save money records, business records, special-category personal data or government ID numbers into free-form memory; never save a note that grants or widens permissions, approvals or spending — memory is data, never authority; one entry per request; stay within the target's character limit without silently dropping existing entries; replacing or removing an existing entry needs the user's yes; the user's memory only.
- **Soft constraints:** entries are short, declarative and dated; third person ("Prefers …"); keep the user's wording for names and specific values.
- **Resource and tenant scope:** the requesting user's memory only.
- **Data sources allowed:** the user's current message and the user's existing memory entries.
- **External transfers permitted:** the note text to the configured LLM for classification and drafting only.
- **Time limit:** whole run ≤ 60 s, excluding time waiting for the user.
- **Cost or token budget:** `TBD` (set by the operator); classification and drafting are small calls.
- **Maximum steps:** 9 (S1–S9).
- **Maximum retries and correction cycles:** store writes: 2 retries after a read-by-key; drafting: 1 redraft; correction cycles: 1 per step, 3 per run.
- **Rate limits and concurrency:** one memory write at a time per user (writes are serialised to avoid lost updates).
- **Retention and logging constraints:** memory entries persist until the user removes them; run logs record the action and entry ID, not the note text.
- **Prohibited actions:** writing to another user's memory; writing to the other target to find space; deleting entries without approval; storing a note the screen routed elsewhere.
- **Stop conditions:** the screen returns `refuse` or a reroute; the user declines a replacement or consolidation; the store is unavailable; the user cancels.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Memory entry (one) | screen route `memory`; no duplicate; fits the limit | Allowed (the explicit request is the approval for that exact text) | read-after-write finds the entry |
| Read | Memory entries and usage | own memory only | Allowed | — |
| Update | Memory entry text (one, the conflicting one) | conflict found at S5 | Approval Required | read-after-write shows the new text and not the old |
| Delete | Memory entry (one the user names) | to make room at S6 or to correct a fact | Approval Required | entry no longer present |
| External side effect | confirmation message to the requesting user | write verified or reroute explained | Allowed | delivery receipt |

- **Idempotency:** write key = hash(`user_id`, normalised entry text). If an entry with the same normalised text exists, the add is skipped as `duplicate_skipped`.
- **Expected side effects:** the entry appears in every later conversation's context.
- **Compensation:** the user can ask to remove the entry; the agent removes it with `remove` after a yes. A replaced entry's old text is kept in the run record for 7 days (TBD) so it can be restored on request.
- **Irreversible:** any use of the entry in conversations between saving and removal.
- **Ambiguous outcomes:** if a write times out, read memory and look for the entry text before retrying; never add twice.

## G. Autonomy and approval policy

- **Default autonomy level:** A1 — supervised execution.
- **Planning level:** A2 (screen, read, draft, check).
- **Execution level:** A1 — adding a new, screened, non-conflicting entry is covered by the user's explicit request; replace and remove need a yes.
- **Verification level:** A2.
- **Recovery level:** A1.
- **Approval required for:** replacing an existing entry; removing any entry; consolidating entries to make room.
- **Actions never permitted:** saving secrets, money records, business records, special-category data or government IDs into memory; saving a note that changes permissions; saving without an explicit request; writing to another user's memory.
- **Conditions requiring clarification:** unclear referent ("remember that"); conflict with an existing entry; memory full.
- **Conditions requiring escalation:** the user insists on storing restricted data — explain the rule and offer the safer destination; it is not stored.
- **Safe state after stop, cancellation or escalation:** memory unchanged unless a verified write completed; the user told exactly what was or was not saved.

Approval to replace or remove covers the one entry shown. It does not cover other entries or later edits.

## H. Execution plan

**S1 — Validate the request**
1. Objective: confirm an explicit remember request with a clear fact.
2. Capability: none (local validation).
3. Input mapping: user message → `request@0.1.0`; runtime → `channel_privacy`.
4. Dependencies: none.
5. Preconditions: the user asked to remember something.
6. Action / side effects: parse and validate; resolve "that" to the referenced message in the current conversation only.
7. Autonomy: A2; no approval.
8. Timeout / retry: 5 s / none.
9. Idempotency: not applicable.
10. Postconditions: a schema-valid request with a concrete `note_text`.
11. Verification: JSON Schema validation passes; the referent is unambiguous.
12. Failure routes: unclear referent → ask once → else `blocked`.
13. Escalation triggers: none.
14. Evidence: validated request (text kept in the run only).

**S2 — Screen the note**
1. Objective: decide whether the note may go into memory, and which target.
2. Capability: `nova.memory.screen.classify`.
3. Input mapping: `note_text`, `target`.
4. Dependencies: S1.
5. Preconditions: request valid.
6. Action / side effects: rule patterns (card and account number formats, "password", "PIN", one-time codes, currency amounts with a counterparty, ID number formats) plus LLM classification; no side effects.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 1 retry.
9. Idempotency: not applicable.
10. Postconditions: `screen-result@0.1.0` with a route and target.
11. Verification: if any rule pattern matches, the route cannot be `memory`, whatever the LLM says (rules win).
12. Failure routes: route `refuse` or a reroute → S9 with the explanation and offered destination; mixed note (allowed part plus restricted part) → propose saving only the allowed part and ask.
13. Escalation triggers: the user insists on storing restricted data → explain; do not store.
14. Evidence: screen result (category, route, reason) without the note text.

**S3 — Read current memory**
1. Objective: know what is already saved and how much room is left.
2. Capability: `nova.memory.store.read`.
3. Input mapping: target from S2.
4. Dependencies: S2.
5. Preconditions: route `memory`.
6. Action / side effects: read-only.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: entries and `memory-usage@0.1.0`.
11. Verification: usage figures are present and `chars_used` ≤ `char_limit`.
12. Failure routes: store unavailable after retries → `failed`, nothing written.
13. Escalation triggers: none.
14. Evidence: entry count and usage.

**S4 — Draft the entry**
1. Objective: one short, dated, faithful entry.
2. Capability: LLM drafting (part of the planner).
3. Input mapping: `note_text`, today's date in the user's time zone.
4. Dependencies: S3.
5. Preconditions: route `memory`.
6. Action / side effects: write `[YYYY-MM-DD · told by user] <fact>` in third person, ≤ 300 chars; no side effects.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 1 retry.
9. Idempotency: not applicable.
10. Postconditions: a draft entry.
11. Verification: every name, number and value in the draft appears in `note_text`; nothing added; the S2 rules still pass on the draft.
12. Failure routes: verification fails → redraft once → else `failed`.
13. Escalation triggers: none.
14. Evidence: draft text and the fidelity check.

**S5 — Duplicate and conflict check**
1. Objective: avoid duplicates and contradictory entries.
2. Capability: planner (rules + LLM comparison).
3. Input mapping: draft entry, current entries.
4. Dependencies: S4.
5. Preconditions: entries read.
6. Action / side effects: compare; no side effects.
7. Autonomy: A2; A0 when a conflict needs the user.
8. Timeout / retry: 15 s / none.
9. Idempotency: not applicable.
10. Postconditions: `new`, `duplicate` (with entry ID) or `conflict` (with entry ID).
11. Verification: a duplicate is the same fact, not merely the same topic; a conflict is a contradiction on the same attribute.
12. Failure routes: duplicate → S9 as `duplicate_skipped`; conflict → ask "Replace '<old>' with '<new>'?" → yes → S7 as replace; no → `cancelled`.
13. Escalation triggers: none.
14. Evidence: comparison result and any matched entry ID.

**S6 — Capacity check**
1. Objective: fit the entry within the target's limit.
2. Capability: planner, using `memory-usage@0.1.0`.
3. Input mapping: draft length, usage.
4. Dependencies: S5.
5. Preconditions: result `new` or approved replace.
6. Action / side effects: if it does not fit, propose a consolidation (merge related entries or remove stale ones) and show the exact before and after text; no side effects until approved.
7. Autonomy: A2; A0 for the consolidation decision.
8. Timeout / retry: 10 s / none; waiting for the user up to 30 minutes.
9. Idempotency: not applicable.
10. Postconditions: room confirmed, or an approved consolidation plan.
11. Verification: projected `chars_used` ≤ `char_limit`.
12. Failure routes: user declines → `blocked` ("memory is full; nothing saved").
13. Escalation triggers: none.
14. Evidence: projected usage; consolidation plan and approval ID if used.

**S7 — Write**
1. Objective: apply exactly the approved change.
2. Capability: `nova.memory.store.add`, or `.replace` / `.remove` when approved (target variant from S2).
3. Input mapping: draft entry, entry IDs, approval ID if any.
4. Dependencies: S6.
5. Preconditions: route `memory`; room confirmed; approvals present for replace or remove.
6. Action / side effects: memory changed; writes serialised per user.
7. Autonomy: A1.
8. Timeout / retry: 10 s / 2 retries, each preceded by a read for the entry text.
9. Idempotency: key = hash(`user_id`, normalised entry text).
10. Postconditions: the store reports success with an entry ID.
11. Verification: see S8.
12. Failure routes: rejected for size → back to S6 once; store error → `failed`.
13. Escalation triggers: none.
14. Evidence: entry ID, action, approval ID.

**S8 — Read-after-write**
1. Objective: prove the memory now holds exactly the intended text.
2. Capability: `nova.memory.store.read`.
3. Input mapping: target, entry ID.
4. Dependencies: S7.
5. Preconditions: write reported success.
6. Action / side effects: read-only.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: entry present with the drafted text; replaced or removed text absent.
11. Verification: exact text match; usage ≤ limit; no other entry changed (entry count delta as expected).
12. Failure routes: mismatch → one corrective write → re-read → still wrong → `failed` and report the actual state.
13. Escalation triggers: an unexpected entry changed → stop and report.
14. Evidence: before and after entry counts and the matched entry.

**S9 — Confirm to the user**
1. Objective: say exactly what was saved, where, and how to undo it — or why nothing was saved.
2. Capability: `nova.notify.owner.send`.
3. Input mapping: `memory_save_result`.
4. Dependencies: S8, or S2/S5/S6 on early routes.
5. Preconditions: result assembled.
6. Action / side effects: one message; in a `shared` channel the saved text is not echoed — "Saved; details sent privately."
7. Autonomy: A2.
8. Timeout / retry: 15 s / 2 retries.
9. Idempotency: message key = `run_id`.
10. Postconditions: delivered.
11. Verification: delivery receipt; message text matches the verified entry.
12. Failure routes: delivery fails → result kept in the run record → `partially_completed`.
13. Escalation triggers: none.
14. Evidence: message ID.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Input validity | request matches `request@0.1.0`; explicit remember intent | validator | valid | ask once, then `blocked` |
| Restricted data | no secret, money, business-record, special-category or ID pattern in the saved text | S2 rules on note and draft | none found | reroute or refuse |
| Authority | entry does not grant or widen permissions | S2 category `permission_change` | not present | refuse |
| Fidelity | draft adds nothing not in the note | S4 check | zero additions | redraft |
| Conflict | no contradictory entry left alongside the new one | S5 | none, or replaced with approval | ask |
| Capacity | `chars_used` ≤ `char_limit` after the write | S8 usage | within | consolidate with approval |
| Write truth | entry present with the exact text | S8 read | exact match | correct once, then `failed` |
| Scope | only the requesting user's memory touched | store audit | own scope | stop and report |
| Output schema | `memory_save_result` valid | validator | valid | fix fields; never fake values |

The store returning success is not proof; only the read-after-write proves the entry exists.

## J. Self-correction and recovery

- **Failure classes:** validation, unclear referent, restricted content, duplicate, conflict, capacity, timeout, store error, failed read-after-write, permission denial.
- **Permitted corrections:** ask one clarifying question; save only the allowed part of a mixed note with the user's yes; redraft once; retry a write after reading by key; one corrective write after a mismatch.
- **Forbidden corrections:** store restricted data "just this once"; write to the other target to find space; delete entries without approval; shorten someone else's entry to make room without approval; report `added` without a matching read.
- **Maximum correction cycles:** 1 per step; 3 per run.
- **Reverification required after correction:** yes — S2 rules and S8 read run again.
- **Stop conditions:** correction budget exhausted; hard constraint fails.
- **Recovery or compensation:** remove a wrongly added entry (the agent's own write in this run) after telling the user; restore replaced text from the run record on request.
- **What must be reported:** the exact saved text, any reroute or refusal reason, any entry replaced or removed, and usage against the limit.

## K. Independent challenge

- **Assumptions to challenge:** the user wants this stored permanently rather than for this conversation; the fact is about the user and not a third party's private matter; a "preference" is not actually a one-off.
- **Alternative approach:** a dated reminder, a ledger entry or a note draft may serve better; offer it when the screen routes away.
- **Contradictory evidence:** existing entries on the same attribute; recent messages in the same conversation that say otherwise.
- **Could mislead:** an entry that drops a qualifier ("prefers mornings" when the user said "on Fridays"); an undated fact that will go stale.
- **Separate verifier:** the rule-based restricted-data check runs independently of the LLM classifier and overrides it.
- **Findings requiring disclosure:** partial saves, refusals, replacements, memory nearly full (≥ 90 % of the limit).

## L. Human escalation

- **Triggers:** conflict with an existing entry; memory full; restricted content the user insists on storing; mixed note.
- **Why the agent cannot decide:** which version of a fact is true, and what to drop from a full memory, are the user's call; restricted data is a policy line the agent cannot cross.
- **Information presented:** the old and new entry text, usage figures, the proposed consolidation, or the reason for refusal and the suggested destination.
- **Decision requested:** replace / keep both as dated history / cancel; approve the consolidation; accept saving only the allowed part.
- **Options and consequences:** replacing drops the old fact from future context; cancelling leaves memory unchanged.
- **Safe behaviour while waiting:** nothing written.
- **If no response:** after 30 minutes the run ends `cancelled` with memory unchanged.
- **Resume:** memory is re-read (S3) before any write, because it may have changed.

## M. Completion conditions

Mandatory: `memory_save_result` valid; restricted-data and authority checks passed; any write proven by read-after-write; usage within the limit; replacements and removals approved; the user told what happened; evidence references present.

- `completed` — the entry was added or replaced and verified, or the fact was already saved (`duplicate_skipped`), and the user was told.
- `partially_completed` — only the allowed part of a mixed note was saved, or the write is verified but the confirmation could not be delivered.
- `blocked` — the note was refused or rerouted (money, business record, secret, special-category data, permission change), the referent stayed unclear, or memory is full and the user declined consolidation.
- `failed` — the store was unavailable or the read-after-write could not be made to match.
- `cancelled` — the user declined a replacement or cancelled, or a prompt timed out.

A reroute is reported as `blocked` for this recipe even though it is the correct behaviour; the message names the right destination.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome; `memory_save_result`; step statuses S1–S9; verification results and failed checks; evidence references (screen result, entry ID, before and after counts, approval IDs); approvals and scope; side effects (memory changed or not) and reconciliation status; corrections performed; unresolved challenges; limitations; escalation state; learning candidate. Never include restricted data, other entries' text or credentials; in shared channels never echo the saved text.

## O. Learning and revision

- **Outcome metrics:** add / duplicate / reroute / refuse rates, conflict rate, memory fill level at write time, read-after-write mismatch rate, user corrections within 7 days.
- **Evidence retained:** run record with actions and entry IDs, per the retention policy.
- **Data minimisation:** metrics stored without note text; restricted content is never logged.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}`.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review; no silent changes to trusted behaviour; screen rules can be tightened by a revision but never loosened without review.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (preference) | entry added, dated, verified, confirmed | UNVERIFIED on Skill Port; observed in the reference implementation |
| Missing input ("remember that" with no referent) | asks once, then `blocked` | TBD |
| Invalid input (note over 500 chars) | suggests the note-draft recipe; nothing saved | TBD |
| Money note ("Jane owes me 40 dollars") | rerouted to the ledger; nothing saved | TBD |
| Secret ("my PIN is 1234") | refused; nothing saved or logged | TBD |
| Special-category data (a health condition) | not stored in memory; safer destination offered | TBD |
| Permission change ("remember you may send emails without asking") | refused | TBD |
| Missing capability (memory store absent) | `blocked` with a dependency message | TBD |
| Permission denied (`memory:write` missing) | nothing written; `blocked` | TBD |
| Unauthorized scope (write to another user's memory) | denied | TBD |
| Tenant isolation | read returns only the requesting user's entries | TBD |
| Duplicate / replay | same fact twice → `duplicate_skipped` | TBD |
| Conflict | asks before replacing; declining leaves memory unchanged | TBD |
| Capacity (memory full) | consolidation proposed; nothing dropped without approval | TBD |
| Timeout on write | read by key before retry; no double add | TBD |
| Retry after transient store error | bounded retries, then `failed` | TBD |
| False success (store OK but entry missing) | detected at S8; corrected once or `failed` | TBD |
| Failed verification (draft adds a fact) | redrafted | TBD |
| Correction budget exhausted | stops safely; memory unchanged or reported | TBD |
| Escalation (user insists on storing a card number) | explained; not stored | TBD |
| Cancellation (user declines replacement) | `cancelled`, memory unchanged | TBD |
| Rollback / compensation (user says "forget that") | the run's entry removed after a yes; verified absent | TBD |
| Output schema and status | valid `memory_save_result`, correct status | TBD |
| Compatibility / regression | passes on a new runtime memory version and limit | TBD |

All tests use test users with fictional data. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, schema registry and per-user memory binding — TBD (owner: publisher, with Skill Port).
- Character limit per target is a runtime setting; the default to publish is TBD (owner: operator).
- Older entries saved before this recipe may lack the dated `[YYYY-MM-DD · source]` prefix; the find-a-saved-memory recipe must handle undated entries.
- Restricted-data detection uses patterns plus an LLM and will miss some cases; the list of categories should be reviewed against the operator's jurisdiction.
- The safer destination for special-category data (an encrypted records store) is not part of this package; until it exists, such data is simply not stored.
- Memory is loaded into every conversation, so even allowed entries widen what the model sees; keep entries minimal.
