# Dismiss an attention item

## A. Identity and ownership

- **Recipe ID:** `nova.inbox.dismiss-an-attention-item` (proposed until assigned by Skill Port)
- **Name:** Dismiss an attention item
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** productivity
- **Tags:** email, attention-list, dismiss, undismiss, soft-flag, reversible, audit
- **Summary:** When the owner dismisses an email from the "needs your attention" list, the agent sets a reversible soft flag on every copy of that email, so it leaves the attention list and the daily digest. The triage record (importance, needs-reply, summary) is kept unchanged, and an undismiss clears the flag.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** Triage results from recipe 08; the attention list used by the dashboard and by recipe 09; an activity (audit) log; an owner interface (reference: a dashboard panel with per-item Dismiss and an undo toast; chat commands are also possible). Store reference: PostgreSQL behind a security-definer dispatcher function. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — reference implementation status: Built. Per-item dismiss and undismiss on the attention panel, with exclusion from both the list and the digest, ran in Nova's reference implementation. Bulk dismiss with confirmation and the audit entry are additions in this recipe. Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.mail.attention` | Capability | The owner's attention list over triaged email | low | UNVERIFIED |
| `nova.mail.attention.resolve` | Operation (read) | Turn item IDs or a selector into Message-ID groups with their current dismiss state | low | UNVERIFIED |
| `nova.mail.attention.dismiss` | Operation (update) | Set the soft dismiss flag on every copy of each Message-ID | low | UNVERIFIED |
| `nova.mail.attention.undismiss` | Operation (update) | Clear the soft dismiss flag on every copy of each Message-ID | low | UNVERIFIED |
| `nova.mail.attention.list` | Operation (read) | Return the current attention list (used to verify the effect) | low | UNVERIFIED |
| `nova.activity.log` | Capability | Append-only audit of agent and owner actions | low | UNVERIFIED |
| `nova.activity.log.append` | Operation (create) | Record who dismissed or undismissed what, and when | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.mail.attention.resolve.by-id` — explicit item IDs from the owner's click. Selected when IDs are given.
- `nova.mail.attention.resolve.by-selector` — a selector from a chat request ("dismiss everything from newsletter@example.com this week"). Read-only; always followed by the confirmation in S3. It never widens beyond the attention list.
- `nova.mail.attention.dismiss.default`, `nova.mail.attention.undismiss.default`, `nova.mail.attention.list.default`, `nova.activity.log.append.default` — only the default variant is supported.

**Bindings:** `binding://nova/mail/attention-resolve`, `binding://nova/mail/attention-dismiss`, `binding://nova/mail/attention-undismiss`, `binding://nova/mail/attention-list`, `binding://nova/activity/append` — all `status: UNVERIFIED`. Reference implementation: dashboard actions call the dispatcher function's `dismiss` and `undismiss` actions; the list and the digest both filter on the flag.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Attention item (view over message copies) | `schema://nova/dismiss-an-attention-item/attention-item@0.1.0` | owner's tenant | read; update (`attention_dismissed_at`, `attention_dismissed_by` only) |
| Triage record | fields on the message copy | owner's tenant | none (must stay unchanged) |
| Audit entry | `schema://nova/dismiss-an-attention-item/audit-entry@0.1.0` | owner's tenant | create |

**Authorization requirements (declared, never a grant):** `mail:attention:read`, `mail:attention:dismiss`, `activity:append`. The actor must be the authenticated owner (or an assistant acting on the owner's explicit request in their own session).

**Policy / trust:** risk class `low` — internal, reversible flag; no provider mailbox change and no third-party effect. The main risk is hiding something important by mistake, addressed by reversibility, confirmation for bulk and agent-built selections, and an audit entry.

## C. Intent and applicability

- **User problem:** An email flagged as important or needing a reply no longer needs the owner (handled elsewhere, irrelevant, false alarm) but keeps showing up.
- **Desired outcome:** The email disappears from the attention list and the digest immediately, its triage stays intact for history and learning, and the owner can bring it back at any time.
- **Trigger or entry conditions:** The owner clicks Dismiss or Undismiss, or asks the assistant to dismiss or restore specific items.
- **Applicable situations:** Any item on the attention list; any previously dismissed item (for undismiss).
- **Out-of-scope situations:** Deleting, archiving or labelling mail at the provider; changing triage results (re-triage is a recipe 08 action); muting a sender for the future (a separate rule recipe); the reply-draft queue (recipe 10 decides whether it honours the flag).
- **Required inputs:** action and items (Section D).
- **Prerequisites:** triage results exist for the items; owner authenticated.
- **Required capabilities:** `nova.mail.attention`, `nova.activity.log` (proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port; every binding is a dependency to resolve before activation.
- **Assumptions:** Verified in the reference implementation: a soft flag, exclusion from list and digest, reversible undismiss, triage untouched. Assumed: copies with the same Message-ID should share the flag because the list shows one row per Message-ID.
- **Ambiguity policy:** Ask. If a chat request matches no item or more than the owner seems to mean, show the matches and ask; never dismiss on a guess.
- **Vedaxi task type:** none — not one of the seven task types supported today.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `action` | enum `dismiss`, `undismiss` | yes | owner | — | missing: ask |
| `item_ids` | array of item IDs, 1–50 | one of `item_ids` / `selector` | owner click | — | unknown or other-tenant ID: reject that ID and report |
| `selector` | object: `from`, `subject_contains`, `received_after`, `importance` | one of `item_ids` / `selector` | owner chat request | — | matches 0: tell the owner; matches > 50: ask to narrow |
| `reason` | enum `handled`, `not_relevant`, `false_alarm`, `other` | no | owner | `other` | — |
| `origin` | enum `owner_click`, `assistant_request` | yes | runtime | — | set by the runtime, never by the caller |

```json
{"$id":"schema://nova/dismiss-an-attention-item/dismiss-request@0.1.0","type":"object","required":["action","origin"],"oneOf":[{"required":["item_ids"]},{"required":["selector"]}],"properties":{"action":{"enum":["dismiss","undismiss"]},"item_ids":{"type":"array","items":{"type":"string"},"minItems":1,"maxItems":50},"selector":{"type":"object","properties":{"from":{"type":"string"},"subject_contains":{"type":"string","maxLength":120},"received_after":{"type":"string","format":"date-time"},"importance":{"enum":["high","normal","low"]}},"minProperties":1},"reason":{"enum":["handled","not_relevant","false_alarm","other"]},"origin":{"enum":["owner_click","assistant_request"]}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `dismiss_result` | `dismiss-result@0.1.0` | `status`, `action`, `groups_changed`, `copies_changed`, `already_in_state` | owner interface (toast with Undo) and run record | keys and counts | re-read list and triage fields (S5) |

```json
{"$id":"schema://nova/dismiss-an-attention-item/dismiss-result@0.1.0","type":"object","required":["status","action","groups_changed","copies_changed","already_in_state"],"properties":{"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},"action":{"enum":["dismiss","undismiss"]},"groups_changed":{"type":"integer","minimum":0},"copies_changed":{"type":"integer","minimum":0},"already_in_state":{"type":"integer","minimum":0},"rejected_ids":{"type":"array","items":{"type":"string"}},"undo_token":{"type":["string","null"]},"audit_ref":{"type":["string","null"]}}}
```

Other schemas:

```json
{"$id":"schema://nova/dismiss-an-attention-item/resolved-items@0.1.0","type":"object","required":["groups"],"properties":{"groups":{"type":"array","maxItems":50,"items":{"type":"object","required":["rfc_message_id","copy_ids","dismissed"],"properties":{"rfc_message_id":{"type":"string"},"copy_ids":{"type":"array","items":{"type":"string"},"minItems":1},"dismissed":{"type":"boolean"},"from":{"type":"string"},"subject":{"type":"string"},"triage_hash":{"type":"string"}}}},"rejected_ids":{"type":"array","items":{"type":"string"}}}}
{"$id":"schema://nova/dismiss-an-attention-item/flag-change@0.1.0","type":"object","required":["rfc_message_ids","actor"],"properties":{"rfc_message_ids":{"type":"array","items":{"type":"string"},"minItems":1,"maxItems":50},"actor":{"type":"string"},"reason":{"enum":["handled","not_relevant","false_alarm","other"]}}}
{"$id":"schema://nova/dismiss-an-attention-item/attention-query@0.1.0","type":"object","properties":{"lookback_hours":{"type":"integer","minimum":1,"maximum":168},"include_dismissed":{"type":"boolean"}}}
{"$id":"schema://nova/dismiss-an-attention-item/attention-item@0.1.0","type":"object","required":["item_id","rfc_message_id","importance","needs_reply","summary","dismissed"],"properties":{"item_id":{"type":"string"},"rfc_message_id":{"type":"string"},"importance":{"enum":["high","normal","low"]},"needs_reply":{"type":"boolean"},"summary":{"type":"string","maxLength":160},"dismissed":{"type":"boolean"},"dismissed_at":{"type":["string","null"],"format":"date-time"}}}
{"$id":"schema://nova/dismiss-an-attention-item/audit-entry@0.1.0","type":"object","required":["kind","actor","target_refs","at"],"properties":{"kind":{"enum":["mail.dismiss","mail.undismiss"]},"actor":{"type":"string"},"origin":{"enum":["owner_click","assistant_request"]},"target_refs":{"type":"array","items":{"type":"string"}},"reason":{"type":"string"},"at":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/dismiss-an-attention-item/audit-receipt@0.1.0","type":"object","required":["audit_ref"],"properties":{"audit_ref":{"type":"string"}}}
```

**Example (fictional)**

Input: `{"action":"dismiss","item_ids":["itm_311"],"reason":"handled","origin":"owner_click"}`

Output: `{"status":"completed","action":"dismiss","groups_changed":1,"copies_changed":2,"already_in_state":0,"rejected_ids":[],"undo_token":"undo_9f2","audit_ref":"act_5521"}` — toast: "Dismissed: Jane Doe — Contract renewal. Undo".

## E. Constraints and budgets

- **Hard constraints:** change only `attention_dismissed_at` and `attention_dismissed_by`; never change triage fields, notified markers, reply-draft fields or provider mail; apply to every copy of the same Message-ID in the tenant and nowhere else; owner's own items only; a selection built by the assistant is shown and confirmed before it is applied; at most 50 Message-IDs per request.
- **Soft constraints:** offer Undo immediately; keep a reason when the owner gives one.
- **Resource and tenant scope:** the owner's attention items.
- **Data sources allowed:** attention list and triage fields (read).
- **External transfers permitted:** none.
- **Time limit:** ≤ 5 seconds for a click; confirmation waits up to 10 minutes.
- **Cost or token budget:** none for clicks; selector parsing from chat may use the assistant's LLM within its normal budget.
- **Maximum steps:** 6 (S1–S6).
- **Maximum retries and correction cycles:** store writes 2 retries; 1 correction cycle.
- **Rate limits and concurrency:** last write wins per Message-ID with a conditional update on the observed state; concurrent dismiss and undismiss are serialised.
- **Retention and logging constraints:** audit entries per the operator's activity-log retention (`TBD`); entries carry keys, not subjects.
- **Prohibited actions:** deleting or archiving mail; deleting triage records; bulk changes without confirmation when `origin = assistant_request` or more than 10 items.
- **Stop conditions:** actor not the owner; items outside the tenant; confirmation declined.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | audit entry | after each change | Allowed | entry readable |
| Read | attention items, triage fields | own tenant | Allowed | — |
| Update | `attention_dismissed_at`, `attention_dismissed_by` on every copy of the selected Message-IDs | owner action; ≤ 50 groups | Allowed for a single owner click; Approval Required (S3 confirmation) for assistant-built or > 10-item selections | list re-read |
| Update | triage, notified, reply-draft fields | — | Not Allowed | before/after hash equal |
| Delete | any record or provider mail | — | Not Allowed | — |
| External side effect | none | — | Not applicable | — |

- **Idempotency:** dismissing an already-dismissed group (or undismissing an active one) is a no-op counted in `already_in_state`; the original timestamp is kept.
- **Expected side effects:** items leave (or return to) the attention list; the next digest reflects the change.
- **Compensation:** undismiss restores the item; the undo token performs it in one step.
- **Irreversible:** nothing; a digest already sent before the dismiss is not recalled.
- **Ambiguous outcomes:** on a write timeout, re-read the group's flag before retrying.

## G. Autonomy and approval policy

- **Default autonomy level:** A1 — supervised execution of the owner's own instruction.
- **Planning level:** A2 (resolve the selection).
- **Execution level:** A1 — single owner clicks apply directly; assistant-built or large selections wait for confirmation.
- **Verification level:** A2.
- **Recovery level:** A2.
- **Approval required for:** applying a selector-based or assistant-initiated selection; any request over 10 items.
- **Actions never permitted:** dismissing on the assistant's own initiative without the owner asking; deleting anything; changing triage.
- **Conditions requiring clarification:** zero or ambiguous matches; selections over 50.
- **Conditions requiring escalation:** repeated write failures.
- **Safe state after stop, cancellation or escalation:** nothing changed, or a fully applied and audited change; never a partial group.

## H. Execution plan

**S1 — Validate the request**
1. Objective: a schema-valid request from the owner.
2. Capability: local validation; runtime identity.
3. Input mapping: click or chat → `dismiss-request@0.1.0`; `origin` set by the runtime.
4. Dependencies: none.
5. Preconditions: authenticated owner session.
6. Action / side effects: validate; no side effects.
7. Autonomy: A2; no approval.
8. Timeout / retry: 2 s / none.
9. Idempotency: not applicable.
10. Postconditions: valid request.
11. Verification: JSON Schema passes; actor is the tenant owner.
12. Failure routes: invalid → ask once (chat) or show an error (click); actor mismatch → `blocked`.
13. Escalation triggers: none.
14. Evidence: validated request (no subjects).

**S2 — Resolve the items**
1. Objective: know exactly which Message-ID groups will change.
2. Capability: `nova.mail.attention.resolve` (variants by-id → by-selector).
3. Input mapping: `item_ids` or `selector`.
4. Dependencies: S1.
5. Preconditions: request valid.
6. Action / side effects: read-only; group copies by Message-ID; record each group's current flag and a hash of its triage fields.
7. Autonomy: A2.
8. Timeout / retry: 5 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: `resolved-items@0.1.0`; foreign or unknown IDs in `rejected_ids`.
11. Verification: every group belongs to the tenant; count ≤ 50.
12. Failure routes: zero groups → tell the owner, `completed` with nothing changed; > 50 → ask to narrow.
13. Escalation triggers: none.
14. Evidence: group keys and triage hashes.

**S3 — Confirm when required**
1. Objective: no surprise bulk changes.
2. Capability: user interaction.
3. Input mapping: resolved groups (sender and subject shown to the owner only).
4. Dependencies: S2.
5. Preconditions: `origin = assistant_request` or more than 10 groups; otherwise skip.
6. Action / side effects: show "Dismiss these 12 emails? (list)" with Confirm / Edit / Cancel.
7. Autonomy: A0 — human decision.
8. Timeout / retry: 10 minutes, then `cancelled`.
9. Idempotency: the confirmation binds the resolved set's hash.
10. Postconditions: confirmation record, or cancellation.
11. Verification: confirmed hash equals the resolved set's hash.
12. Failure routes: Edit → S2; Cancel or timeout → `cancelled`.
13. Escalation triggers: none.
14. Evidence: confirmation ID and hash.

**S4 — Apply the flag change**
1. Objective: change the flag on every copy, atomically per request.
2. Capability: `nova.mail.attention.dismiss` or `nova.mail.attention.undismiss`.
3. Input mapping: Message-IDs, actor, reason → `flag-change@0.1.0`.
4. Dependencies: S2 (and S3 when required).
5. Preconditions: each group's flag still matches what S2 observed, or is already in the target state.
6. Action / side effects: dismiss sets `attention_dismissed_at = now` and `attention_dismissed_by = actor` where empty; undismiss clears both.
7. Autonomy: A1.
8. Timeout / retry: 5 s / 2 retries after re-reading.
9. Idempotency: target-state no-ops counted, timestamps preserved.
10. Postconditions: `groups_changed + already_in_state` equals resolved groups.
11. Verification: S5.
12. Failure routes: conflict (state changed since S2) → re-resolve once; store error → `failed`, nothing partial (single transaction).
13. Escalation triggers: repeated store errors.
14. Evidence: change counts.

**S5 — Verify the effect and the triage record**
1. Objective: prove the item left (or returned to) the list and triage stayed intact.
2. Capability: `nova.mail.attention.list`; `nova.mail.attention.resolve` (for hashes).
3. Input mapping: Message-IDs.
4. Dependencies: S4.
5. Preconditions: S4 returned.
6. Action / side effects: read-only.
7. Autonomy: A2.
8. Timeout / retry: 5 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: verification result.
11. Verification: dismissed groups absent from the list (and from the digest's unnotified view); undismissed groups present again if still within the window; triage hashes equal to S2.
12. Failure routes: triage hash changed → report a defect and escalate; list mismatch → retry the read once, then `partially_completed`.
13. Escalation triggers: triage changed.
14. Evidence: list snapshot keys and hash comparison.

**S6 — Audit and respond**
1. Objective: an audit trail and immediate feedback with Undo.
2. Capability: `nova.activity.log.append`.
3. Input mapping: kind, actor, origin, target Message-IDs, reason.
4. Dependencies: S5.
5. Preconditions: a change happened (no audit for pure no-ops).
6. Action / side effects: one audit entry; return the result with an undo token (undo = the opposite action on the same groups).
7. Autonomy: A2.
8. Timeout / retry: 5 s / 2 retries.
9. Idempotency: entry key = request ID.
10. Postconditions: audit reference returned.
11. Verification: entry readable by reference.
12. Failure routes: audit write fails → keep the change, return `partially_completed`, retry the entry asynchronously.
13. Escalation triggers: none.
14. Evidence: audit reference.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Actor | actor is the tenant owner | session identity | match | `blocked` |
| Scope | every group in the tenant; ≤ 50 | S2 | true | reject IDs / ask to narrow |
| Confirmation | required confirmations present and hash-bound | S3 record | present | stop before S4 |
| Field limit | only the two dismiss fields changed | before/after row diff | true | escalate a defect |
| Triage intact | triage hash unchanged | S2 vs S5 | equal | escalate a defect |
| List effect | list and digest view reflect the action | S5 | as expected | retry read; else `partially_completed` |
| Copies | every copy of the Message-ID carries the same flag | S5 | all equal | re-apply once |
| Audit | entry exists for every change | S6 | present | async retry; disclose |

## J. Self-correction and recovery

- **Failure classes:** invalid request, foreign IDs, ambiguous selector, confirmation declined or timed out, state conflict, store error, audit failure.
- **Permitted corrections:** re-resolve after a conflict; retry reads and writes; retry the audit entry asynchronously.
- **Forbidden corrections:** widening a selector to "make it match"; applying without a required confirmation; touching triage fields.
- **Maximum correction cycles:** 1 per request.
- **Reverification required after correction:** yes — S5 after any re-apply.
- **Stop conditions:** actor mismatch; confirmation declined.
- **Recovery or compensation procedure:** undo token or a later undismiss.
- **What must be reported to the user:** what changed, what was rejected, and the Undo option.

## K. Independent challenge

- **Assumptions to challenge:** that the owner meant every match of a chat selector; that "handled" items truly need nothing more.
- **Alternative approach:** if the owner dismisses the same sender repeatedly, suggest a sender rule or a re-triage instead (never apply it automatically).
- **Contradictory evidence:** an item dismissed and then replied to by the owner suggests a misclick; surface the pattern, do nothing automatically.
- **Could mislead:** a dismissed item reappearing because a new copy arrived with a different Message-ID; the result explains this.
- **Separate verifier:** S5 reads the list through the same view the digest uses, not through the write path.
- **Findings requiring disclosure:** rejected IDs; groups already in the target state.

## L. Human escalation

- **Triggers:** confirmation required (S3); triage hash changed; repeated store errors.
- **Why the agent cannot decide:** bulk hiding of mail is the owner's choice; a changed triage record indicates a defect.
- **Information presented:** the matched emails (owner-only view), counts, the defect evidence.
- **Decision requested:** confirm, edit or cancel; for defects, acknowledge and pause bulk actions.
- **Options and consequences:** confirming hides the items from the list and digest; cancelling changes nothing.
- **Safe behaviour while waiting:** nothing applied.
- **If no response:** `cancelled` after 10 minutes.
- **Resume:** a new request re-runs S1–S2.

## M. Completion conditions

Mandatory: result valid; only the dismiss fields changed; triage unchanged; list effect verified; audit entry present for every change; rejected IDs disclosed.

- `completed` — all resolved groups are in the target state, verified and audited (including the case where nothing matched and the owner was told).
- `partially_completed` — the change applied but the audit entry or the list verification is pending.
- `blocked` — the actor or scope check failed.
- `failed` — the store rejected the change; nothing changed.
- `cancelled` — the owner declined or the confirmation timed out.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome ("Dismissed 1 email (2 copies). Undo"); `dismiss-result@0.1.0`; step statuses S1–S6; verification results and failed checks; evidence references (audit reference, confirmation ID); approvals and scope; side effects (flags changed) and reconciliation status; corrections; unresolved challenges; limitations; escalation state; learning candidate. Never include bodies; subjects only in the owner's own interface.

## O. Learning and revision

- **Outcome metrics:** dismiss rate per importance level, undismiss rate (misclick or regret signal), time from digest to dismiss, top dismissed senders (counts only) as feedback to recipe 08.
- **Evidence retained:** audit entries, per the retention policy.
- **Data minimisation:** metrics use counts and keys; no subjects.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}`.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (dismiss one fictional item) | leaves list and digest; triage unchanged; Undo works | UNVERIFIED on Skill Port; observed in the reference implementation |
| Undismiss | returns to the list if still in the window | UNVERIFIED on Skill Port; observed in the reference implementation |
| Invalid input (unknown action) | rejected | TBD |
| Missing input (no items, no selector) | asks | TBD |
| Missing capability (audit log absent) | change applied, `partially_completed`, disclosed — or `blocked` if the operator requires audit | TBD |
| Permission denied (`mail:attention:dismiss` missing) | `blocked` | TBD |
| Unauthorized scope (another tenant's item ID) | rejected ID, nothing changed for it | TBD |
| Tenant isolation | copies in another tenant with the same Message-ID unchanged | TBD |
| Copies | both mailbox copies flagged | TBD |
| Duplicate / replay (dismiss twice) | second is a no-op; timestamp kept | TBD |
| Concurrent dismiss and undismiss | serialised; final state matches the last confirmed action | TBD |
| Timeout on write | re-read before retry; no double audit | TBD |
| False success (write OK, list still shows item) | S5 catches it; `partially_completed` | TBD |
| Failed verification (triage hash changes) | escalated as a defect | TBD |
| Bulk selector from chat (12 matches) | confirmation shown before applying | TBD |
| Correction budget exhausted | stops with the honest status | TBD |
| Escalation (selector > 50) | asks to narrow | TBD |
| Cancellation (owner declines confirmation) | nothing changed | TBD |
| Compensation (Undo token) | restores the previous state | TBD |
| Output schema and status | valid `dismiss-result` | TBD |
| Compatibility / regression (digest still excludes dismissed after a schema change) | excluded | TBD |

All tests use fictional mail in a sandbox tenant. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs and the owner-interface contract for Undo — TBD (owner: publisher, with Skill Port).
- Whether the reply-draft queue (recipe 10) honours the dismiss flag — open; recommended default: yes, configurable.
- Whether a dismiss should expire (for example, reappear if the sender writes again in the same thread) — open design decision.
- Audit retention — TBD (owner: operator).
- A digest already delivered cannot be recalled; dismiss affects future digests only.
