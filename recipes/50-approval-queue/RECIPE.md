# Approval queue

## A. Identity and ownership

- **Recipe ID:** `nova.ops.approval-queue` (proposed until assigned by Skill Port)
- **Name:** Approval queue
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** operations
- **Tags:** human-in-the-loop, approval-gateway, single-use, expiry, scope-binding, audit, outbound-actions
- **Summary:** A gateway that any automation calls before an outbound action. The owner approves or rejects one specific action within an expiry window; the decision is recorded with who, when and the exact scope; and an approval can be used once, for that action only.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** A transactional database for approval records; an authenticated decision endpoint on the owner's dashboard; an owner notification channel; the activity log (`nova.ops.activity-log`) for the audit trail; a scheduler for the expiry sweep. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — the core flow (request → chat message with approve/deny links → decision → audit entry, plus status checks and an expiry sweep) runs in Nova's reference implementation; not every control described here has evidence yet (Section P lists the tests still needed). Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.approvals.queue` | Capability | Hold approval records through their lifecycle | high | UNVERIFIED |
| `nova.approvals.queue.request` | Operation (create) | Open a pending approval for one action scope | medium | UNVERIFIED |
| `nova.approvals.queue.cancel` | Operation (update) | Requester withdraws a pending request | low | UNVERIFIED |
| `nova.approvals.queue.status` | Operation (read) | Return the state for a presented scope | low | UNVERIFIED |
| `nova.approvals.queue.consume` | Operation (update) | Atomically turn `approved` into `consumed` (single use) | high | UNVERIFIED |
| `nova.approvals.queue.record_outcome` | Operation (update) | Record whether the approved action succeeded | low | UNVERIFIED |
| `nova.approvals.queue.expire_sweep` | Operation (update) | Expire pending and unused approvals past their deadlines | low | UNVERIFIED |
| `nova.approvals.queue.list_pending` | Operation (read) | List pending approvals for the dashboard and digests | low | UNVERIFIED |
| `nova.approvals.decision` | Capability | Let an authenticated human decide | high | UNVERIFIED |
| `nova.approvals.decision.view` | Operation (read) | Render the confirmation page for a decision token | low | UNVERIFIED |
| `nova.approvals.decision.decide` | Operation (update) | Approve or reject, human principal only | high | UNVERIFIED |
| `nova.approvals.prompt` | Capability | Ask the owner | low | UNVERIFIED |
| `nova.approvals.prompt.send` | Operation (external side effect, owner only) | Deliver the approval prompt | low | UNVERIFIED |
| `nova.activity.log` | Capability (dependency, recipe 51) | Append-only audit trail | low | UNVERIFIED |
| `nova.activity.log.append` | Operation (create) | Write one audit entry in the same transaction as the state change | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.approvals.prompt.send.chat-links` — **default.** A plain-text chat message with the summary, expiry and two links (Approve / Reject). Each link carries a single-use decision token and opens a confirmation page; opening the link changes nothing, so chat link previews cannot approve.
- `nova.approvals.prompt.send.chat-buttons` — inline chat buttons. The platform callback must be verified as genuine and its user ID must equal the configured owner. Eligible for `low` and `medium` risk only.
- `nova.approvals.prompt.send.dashboard` — no message; the item waits in the dashboard's pending list. Always available, and required for `high` and `critical` risk: the decision must come from a signed-in owner session.
  Fallback: if the chat channel fails, the prompt falls back to `dashboard` (the stricter path). There is never a fallback from `dashboard` to a chat variant for high-risk items.
- All other operations — only the default variant is supported.

**Bindings:** `binding://nova/approvals/request`, `binding://nova/approvals/cancel`, `binding://nova/approvals/status`, `binding://nova/approvals/consume`, `binding://nova/approvals/outcome`, `binding://nova/approvals/expire`, `binding://nova/approvals/pending`, `binding://nova/approvals/view`, `binding://nova/approvals/decide`, `binding://nova/approvals/prompt`, `binding://nova/activity/append` — all `status: UNVERIFIED`. Reference implementation: database functions (request, decide, status, expire) with role-scoped grants; the request triggers a webhook on the automation hub that sends the chat prompt; the decision route lives on the owner's dashboard.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Approval | `schema://nova/approval-queue/approval@0.1.0` | the owner's tenant | create, read, update (state machine only) |
| Action type | `schema://nova/approval-queue/action-type@0.1.0` | operator-registered | read |
| Decision token | stored as a SHA-256 hash only | one approval and one decision | create, read (hash compare), update (spent) |
| Audit entry | `schema://nova/approval-queue/audit-entry@0.1.0` | the activity log | create |
| Prompt message | `schema://nova/approval-queue/prompt@0.1.0` | owner channel | create |

**Authorization requirements (declared, never a grant):** requesting automations: `approvals:request`, `approvals:status`, `approvals:consume`, `approvals:outcome`, `approvals:cancel` (own requests only). Human decision endpoint: `approvals:decide` — **never granted to any automation or agent identity**. Scheduler: `approvals:expire`. Dashboard: `approvals:read`. Audit: `activity:append`. Secrets by reference: `secret_ref://owner-channel`, `secret_ref://chat-callback-verification`.

**Policy / trust:** risk class `high` — this is the control that stands between automations and third parties. Requests, summaries and parameters from callers are untrusted: they cannot select the approver, extend expiry beyond the action type's maximum or lower the risk class.

## C. Intent and applicability

- **User problem:** The assistant drafts emails, messages, bookings and posts; the owner wants nothing to reach another person until they say yes — and wants proof afterwards of exactly what was approved.
- **Desired outcome:** One uniform yes/no gate: a clear prompt, a decision bound to one exact action, an expiry, single use, and a permanent record.
- **Trigger or entry conditions:** An automation or recipe about to perform a registered outbound action calls `request`; the owner decides; the automation checks `status` and calls `consume` immediately before acting.
- **Applicable situations:** Email or message sends, publishing a post, sending a calendar invite, placing a call, any registered action type with an external side effect.
- **Out-of-scope situations:** Payments and money movement (need a dedicated, stronger flow); actions without a registered action type; standing or blanket approvals ("approve everything from this workflow"); approvals by anyone other than the configured owner; the outbound action itself (performed by the caller).
- **Required inputs:** `approval-request@0.1.0` (see Section D).
- **Prerequisites:** Action types registered by the operator; owner channel; dashboard with owner sign-in; activity log available.
- **Required capabilities:** `nova.approvals.queue`, `nova.approvals.decision`, `nova.approvals.prompt`, `nova.activity.log` (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port; every binding is a dependency to resolve before activation.
- **Assumptions:** Verified in the reference implementation: the core flow listed in Section A. Not every control in this recipe has evidence yet; all of them are mandatory before activation.
- **Ambiguity policy:** Stop. An unregistered action type, a missing scope or an unclear summary is rejected back to the caller; the owner is never asked to approve something vague.
- **Vedaxi task type:** Not applicable — this recipe does not map to any of the seven task types Vedaxi supports today.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `request_key` | string 8–128 chars, unique per caller | yes | caller | — | missing → reject |
| `action_type` | registered ID, e.g. `email.send` | yes | caller | — | unregistered → reject |
| `target` | `{kind, display, ref}` | yes | caller; may be personal | — | missing → reject |
| `parameters` | object (canonicalised and hashed; not stored) | yes | caller; may be personal | — | not canonicalisable → reject |
| `summary` | string 10–300 chars, plain text | yes | caller; shown to the owner | — | missing or too long → reject |
| `preview` | object of ≤ 10 display fields, each ≤ 200 chars | no | caller | `{}` | over limit → truncated, disclosed |
| `expires_in_minutes` | integer 5–10080 | no | caller | 1440, capped by action type | over cap → capped |
| `use_within_minutes` | integer 5–1440 | no | caller | 60 | out of range → clamped |

```json
{"$id":"schema://nova/approval-queue/approval-request@0.1.0","type":"object","additionalProperties":false,
 "required":["request_key","action_type","target","parameters","summary"],
 "properties":{"request_key":{"type":"string","minLength":8,"maxLength":128},"action_type":{"type":"string","pattern":"^[a-z]+\\.[a-z_]+$"},
 "target":{"type":"object","required":["kind","display"],"properties":{"kind":{"enum":["email","phone","chat","calendar","post","other"]},"display":{"type":"string","maxLength":120},"ref":{"type":"string","maxLength":200}}},
 "parameters":{"type":"object"},"summary":{"type":"string","minLength":10,"maxLength":300},
 "preview":{"type":"object","maxProperties":10,"additionalProperties":{"type":"string","maxLength":200}},
 "expires_in_minutes":{"type":"integer","minimum":5,"maximum":10080},"use_within_minutes":{"type":"integer","minimum":5,"maximum":1440}}}
```

The requester identity comes from the caller's authenticated context, never from the request body.

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `approval` | object `schema://nova/approval-queue/approval@0.1.0` | `approval_id`, `state`, `scope_hash`, `expires_at` | caller, dashboard | personal (target display); owner and caller only | state machine checks (Section I) |
| `consume-result` | `schema://nova/approval-queue/consume-result@0.1.0` | `approval_id`, `granted` | caller | low | atomic transition |

```json
{"$id":"schema://nova/approval-queue/approval@0.1.0","type":"object","required":["approval_id","request_key","action_type","scope_hash","state","risk_class","requested_by","requested_at","expires_at","version"],
 "properties":{"approval_id":{"type":"string","pattern":"^apr_[a-z0-9]{12,32}$"},"request_key":{"type":"string"},"action_type":{"type":"string"},
 "target_display":{"type":"string"},"summary":{"type":"string"},"preview":{"type":"object"},"scope_hash":{"type":"string","pattern":"^[a-f0-9]{64}$"},
 "risk_class":{"enum":["low","medium","high","critical"]},"state":{"enum":["pending","approved","rejected","expired","cancelled","consumed"]},
 "requested_by":{"type":"string"},"requested_at":{"type":"string","format":"date-time"},"expires_at":{"type":"string","format":"date-time"},
 "use_by":{"type":["string","null"],"format":"date-time"},"decided_by":{"type":["string","null"]},"decided_at":{"type":["string","null"],"format":"date-time"},
 "decision_channel":{"type":["string","null"],"enum":[null,"chat-links","chat-buttons","dashboard"]},"decision_note":{"type":["string","null"],"maxLength":300},
 "consumed_at":{"type":["string","null"],"format":"date-time"},"outcome":{"type":["string","null"],"enum":[null,"executed_ok","executed_failed","outcome_unknown"]},
 "outcome_at":{"type":["string","null"],"format":"date-time"},"version":{"type":"integer","minimum":1}}}
```

`scope_hash` = SHA-256 of the canonical JSON (sorted keys, no insignificant whitespace) of `{action_type, target.kind, target.ref, parameters}`.

**Other schemas** (compact JSON Schema):

```json
{"$id":"schema://nova/approval-queue/action-type@0.1.0","type":"object","required":["action_type","risk_class","max_expiry_minutes","allowed_requesters"],"properties":{"action_type":{"type":"string"},"risk_class":{"enum":["low","medium","high","critical"]},"max_expiry_minutes":{"type":"integer","minimum":5,"maximum":10080},"allowed_requesters":{"type":"array","items":{"type":"string"}},"requires_session":{"type":"boolean"}}}
{"$id":"schema://nova/approval-queue/scope-check@0.1.0","type":"object","required":["approval_id","action_type","target","parameters"],"properties":{"approval_id":{"type":"string"},"action_type":{"type":"string"},"target":{"type":"object"},"parameters":{"type":"object"}}}
{"$id":"schema://nova/approval-queue/consume-result@0.1.0","type":"object","required":["approval_id","granted"],"properties":{"approval_id":{"type":"string"},"granted":{"type":"boolean"},"grant_id":{"type":["string","null"]},"reason":{"type":["string","null"],"enum":[null,"not_approved","expired","already_consumed","scope_mismatch","not_requester"]}}}
{"$id":"schema://nova/approval-queue/decision@0.1.0","type":"object","required":["approval_id","decision","token","csrf"],"properties":{"approval_id":{"type":"string"},"decision":{"enum":["approve","reject"]},"token":{"type":"string","minLength":22},"csrf":{"type":"string"},"note":{"type":"string","maxLength":300}}}
{"$id":"schema://nova/approval-queue/decision-view@0.1.0","type":"object","required":["approval_id","state","summary","decision"],"properties":{"approval_id":{"type":"string"},"state":{"type":"string"},"summary":{"type":"string"},"target_display":{"type":"string"},"preview":{"type":"object"},"expires_at":{"type":"string","format":"date-time"},"decision":{"enum":["approve","reject"]},"csrf":{"type":"string"},"session_required":{"type":"boolean"}}}
{"$id":"schema://nova/approval-queue/outcome@0.1.0","type":"object","required":["approval_id","grant_id","outcome"],"properties":{"approval_id":{"type":"string"},"grant_id":{"type":"string"},"outcome":{"enum":["executed_ok","executed_failed","outcome_unknown"]},"external_ref":{"type":["string","null"],"maxLength":200}}}
{"$id":"schema://nova/approval-queue/approval-ref@0.1.0","type":"object","required":["approval_id"],"properties":{"approval_id":{"type":"string"},"reason":{"type":"string","maxLength":200}}}
{"$id":"schema://nova/approval-queue/sweep-request@0.1.0","type":"object","required":["now"],"properties":{"now":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/approval-queue/sweep-result@0.1.0","type":"object","required":["expired_pending","expired_unused","run_at"],"properties":{"expired_pending":{"type":"integer","minimum":0},"expired_unused":{"type":"integer","minimum":0},"run_at":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/approval-queue/pending-query@0.1.0","type":"object","properties":{"cursor":{"type":["string","null"]},"limit":{"type":"integer","minimum":1,"maximum":100,"default":30},"risk_class":{"enum":["low","medium","high","critical"]}}}
{"$id":"schema://nova/approval-queue/pending-list@0.1.0","type":"object","required":["items"],"properties":{"items":{"type":"array","items":{"$ref":"schema://nova/approval-queue/approval@0.1.0"}},"next_cursor":{"type":["string","null"]}}}
{"$id":"schema://nova/approval-queue/prompt@0.1.0","type":"object","required":["approval_id","variant","text"],"properties":{"approval_id":{"type":"string"},"variant":{"enum":["chat-links","chat-buttons","dashboard"]},"text":{"type":"string","maxLength":800}}}
{"$id":"schema://nova/approval-queue/delivery@0.1.0","type":"object","required":["approval_id","delivered"],"properties":{"approval_id":{"type":"string"},"delivered":{"type":"boolean"},"message_id":{"type":["string","null"]}}}
{"$id":"schema://nova/approval-queue/audit-entry@0.1.0","type":"object","required":["source","action","entity_id","actor","status"],"properties":{"source":{"const":"approvals"},"action":{"enum":["approval_requested","approval_approved","approval_rejected","approval_expired","approval_cancelled","approval_consumed","approval_outcome"]},"entity_id":{"type":"string"},"actor":{"type":"string"},"status":{"enum":["ok","failed"]},"title":{"type":"string","maxLength":200},"meta":{"type":"object","properties":{"action_type":{"type":"string"},"risk_class":{"type":"string"},"scope_hash_prefix":{"type":"string","maxLength":12},"channel":{"type":"string"}},"additionalProperties":false}}}
```

Notes: `scope-check@0.1.0`: the caller re-presents the full scope; the queue recomputes the hash.

**Example (fictional)**

Request: `{"request_key":"followup-0912-jane","action_type":"email.send","target":{"kind":"email","display":"Jane Doe <jane@example.com>","ref":"jane@example.com"},"parameters":{"subject":"Thursday recap","body_digest":"3f9a…","thread_ref":"thr_204"},"summary":"Send the Thursday recap email to Jane Doe (reply in existing thread).","preview":{"subject":"Thursday recap","first_line":"Thanks for the time today — here is the recap."},"expires_in_minutes":720}`

After approval: `{"approval_id":"apr_7k2m9q4x1z8w","state":"approved","scope_hash":"<64 hex>","risk_class":"medium","requested_by":"automation:recap-drafter","decided_by":"owner:user_01","decided_at":"2026-11-02T15:04:11Z","decision_channel":"chat-links","use_by":"2026-11-02T16:04:11Z","version":2}`; then `consume` → `{"approval_id":"apr_7k2m9q4x1z8w","granted":true,"grant_id":"gr_51c0"}`; a second `consume` → `{"granted":false,"reason":"already_consumed"}`.

## E. Constraints and budgets

- **Hard constraints:** only a human owner principal can decide — no automation or agent identity holds `approvals:decide`, and a requester can never approve its own request; an approval covers exactly one `scope_hash`; any change to target or parameters needs a new approval; an approval is consumed at most once and only by the identity that requested it; expired, rejected or cancelled approvals can never become approved; opening a link changes nothing — decisions happen only through an explicit confirmation (POST with CSRF protection); high and critical risk require a signed-in owner session; decision tokens are ≥ 128-bit random, stored only as hashes, single-use and expire with the approval; every state change and its audit entry commit in the same transaction; no secrets or raw payloads in summaries, previews or audit entries.
- **Soft constraints:** prompts readable in one screen; the summary states the recipient and the effect in plain words.
- **Resource and tenant scope:** the owner's approvals and registered action types.
- **Data sources allowed:** the caller's request; the action-type registry.
- **External transfers permitted:** the prompt text to the owner channel provider.
- **Time limit:** request ≤ 5 s; decision request ≤ 5 s; pending window and use window as configured.
- **Cost or token budget:** no LLM is used.
- **Maximum steps:** 10 (S1–S10), spread across the lifecycle.
- **Maximum retries and correction cycles:** request and prompt: 2 retries; consume: 0 retries after an ambiguous result (read status instead); correction cycles: 1.
- **Rate limits and concurrency:** at most 20 pending approvals per requester (back-pressure); at most 5 decision attempts per token per minute.
- **Retention and logging constraints:** approval records and audit entries kept for the operator's audit period (`TBD`, suggested ≥ 1 year); logs never contain tokens or parameters.
- **Prohibited actions:** blanket or standing approvals; approving on the owner's behalf; reusing an approval for a similar action; extending an expired approval; sending the prompt to anyone but the owner.
- **Stop conditions:** unregistered action type; requester not allowed for the action type; owner channel and dashboard both unavailable; audit append fails (the transaction rolls back).

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Approval (`pending`) + two decision tokens | registered action type; requester allowed; unique `request_key` | Allowed | record and audit entry read back |
| Create | Prompt message | approval `pending` | Allowed (owner only) | delivery receipt |
| Create | Audit entry | every state change, same transaction | Allowed | entry exists for each transition |
| Read | Approval (own requests; owner sees all), pending list, action types | own tenant | Allowed | — |
| Update | Approval `state` pending→approved/rejected; `decided_by`, `decided_at`, `decision_channel`, `decision_note`, `use_by` | valid token + confirmation; owner principal; not expired; version CAS | Human decision only | read-back + audit |
| Update | `state` approved→consumed; `consumed_at`, `grant_id` | requester identity; scope hash matches; before `use_by` | Allowed (this is the use of the approval) | atomic row count = 1 |
| Update | `outcome`, `outcome_at` | state `consumed`; matching `grant_id`; set once | Allowed | read-back |
| Update | `state` pending→cancelled | requester's own pending request | Allowed | read-back |
| Update | `state` → expired | deadline passed | Allowed (sweep) | read-back |
| Delete | approvals, tokens, audit entries | — | Not Allowed (retention handled by the audit log's own policy) | — |
| External side effect | the approved action itself | — | Not performed here (the caller performs it after `consume`) | caller's outcome record |
| External side effect | prompt to the owner | pending approval | Allowed | delivery receipt |

- **Idempotency:** `request_key` is unique per requester — a repeat returns the existing approval. `consume` is a conditional update (`state='approved' AND scope_hash=… AND use_by>now()`) affecting at most one row. Decisions use version compare-and-swap; a second decision on a decided approval returns its current state.
- **Expected side effects:** an owner prompt; state changes with audit entries.
- **Compensation:** an approval decided by mistake can be revoked by the owner while still `approved` (→ `rejected`, audited); once `consumed`, the caller's own compensation applies.
- **Irreversible:** a consumed approval; the external action the caller performed.
- **Ambiguous outcomes:** if `consume` times out, the caller reads `status` before doing anything; `consumed` with no outcome means the caller must reconcile its side effect, never consume again.

## G. Autonomy and approval policy

- **Default autonomy level:** A1 — supervised execution.
- **Planning level:** A2 (validate, policy check, compose the prompt).
- **Execution level:** A1 for request, prompt, consume and sweep; **A0 for the decision itself** (human only).
- **Verification level:** A2.
- **Recovery level:** A1 — no re-consume; a new approval is needed after any failure.
- **Approval required for:** every registered outbound action (that is the purpose); registering or changing an action type (operator approval); revoking an approval (owner).
- **Actions never permitted:** an agent or automation deciding; blanket approvals; reusing or extending approvals; lowering an action type's risk class at request time; prompting anyone but the owner.
- **Conditions requiring clarification:** none from the owner — vague requests are rejected to the caller.
- **Conditions requiring escalation:** repeated invalid tokens (possible link leak); audit append failing; owner unreachable on both chat and dashboard.
- **Safe state after stop, cancellation or escalation:** the approval stays `pending` until expiry, or becomes `expired`/`cancelled`; nothing is consumed; no outbound action can proceed.

Each approval is specific to one action and scope and is single-use. It confers no ongoing permission.

## H. Execution plan

**S1 — Validate the request**
1. Objective: accept only precise, well-formed requests.
2. Capability: `nova.approvals.queue.request` (validation stage).
3. Input mapping: caller payload → `approval-request@0.1.0`; requester identity from the authenticated context.
4. Dependencies: none.
5. Preconditions: caller authenticated with `approvals:request`.
6. Action / side effects: schema validation; canonicalise `parameters`; compute `scope_hash`; scan `summary` and `preview` for secrets (token, key and password patterns).
7. Autonomy: A2.
8. Timeout / retry: 2 s / none.
9. Idempotency: deterministic.
10. Postconditions: a valid request and its `scope_hash`.
11. Verification: schema passes; secret scan clean.
12. Failure routes: invalid → reject to the caller with field paths; secret found → reject (`secret_in_summary`), never stored.
13. Escalation triggers: none.
14. Evidence: request key, validation result.

**S2 — Policy check**
1. Objective: confirm this requester may ask for this action type.
2. Capability: action-type registry read (`action-type@0.1.0`).
3. Input mapping: `action_type`, requester identity.
4. Dependencies: S1.
5. Preconditions: registry reachable.
6. Action / side effects: read-only; derive `risk_class`, cap `expires_in_minutes`, decide the prompt variant (high/critical → `dashboard`), check the requester's pending count.
7. Autonomy: A2.
8. Timeout / retry: 2 s / 1 retry.
9. Idempotency: read-only.
10. Postconditions: risk class, effective expiry, prompt variant.
11. Verification: requester is in `allowed_requesters`; pending count ≤ 20.
12. Failure routes: unregistered or not allowed → reject; over back-pressure limit → reject with `too_many_pending`.
13. Escalation triggers: a requester repeatedly asking for unregistered types → operator notice.
14. Evidence: policy decision.

**S3 — Create the approval record**
1. Objective: a durable pending approval with its audit entry.
2. Capability: `nova.approvals.queue.request`; `nova.activity.log.append`.
3. Input mapping: validated request + policy → `approval@0.1.0` (`state=pending`, `version=1`).
4. Dependencies: S2.
5. Preconditions: `request_key` unused by this requester (else return the existing record).
6. Action / side effects: one transaction: insert the approval, insert two token hashes (approve, reject), append `approval_requested` to the audit log.
7. Autonomy: A1.
8. Timeout / retry: 5 s / 1 retry (safe: unique `request_key`).
9. Idempotency: unique (`requested_by`, `request_key`).
10. Postconditions: approval `pending`; raw tokens held in memory only for S4.
11. Verification: read-back of the record and the audit entry.
12. Failure routes: transaction fails → nothing exists; the caller is told to retry.
13. Escalation triggers: audit append failing repeatedly → operator notice.
14. Evidence: approval ID, audit entry ID.

**S4 — Send the approval prompt**
1. Objective: put a clear decision in front of the owner.
2. Capability: `nova.approvals.prompt.send` (variant from S2).
3. Input mapping: summary, target display, preview, expiry, links to the confirmation page (chat-links) or buttons (chat-buttons).
4. Dependencies: S3.
5. Preconditions: approval `pending`.
6. Action / side effects: plain-text message to the owner only; raw tokens appear only inside the links and are then discarded from memory.
7. Autonomy: A1.
8. Timeout / retry: 10 s / 2 retries; then fall back to `dashboard`.
9. Idempotency: one prompt per approval ID (resend only on explicit owner request, with fresh tokens and the old ones revoked).
10. Postconditions: a message ID, or the item visible in the dashboard list.
11. Verification: delivery receipt or dashboard visibility.
12. Failure routes: chat fails → dashboard only, disclosed to the caller.
13. Escalation triggers: chat and dashboard both unavailable → `blocked`; the approval expires unused.
14. Evidence: message ID, variant used.

**S5 — Owner decides**
1. Objective: a deliberate, authenticated human decision bound to this approval.
2. Capability: `nova.approvals.decision.view`, then `nova.approvals.decision.decide`.
3. Input mapping: token from the link → view; confirmation form → `decision@0.1.0`.
4. Dependencies: S4.
5. Preconditions: token hash matches an unspent token for this approval; approval `pending`; not expired; owner session present when `session_required`; chat-button callbacks verified and from the owner's user ID.
6. Action / side effects: GET renders the summary, preview and expiry with an explicit Approve/Reject button and changes nothing; POST (CSRF-checked) runs one transaction: compare-and-swap `pending → approved|rejected`, set `decided_by`, `decided_at`, `decision_channel`, `use_by` (approve only), mark both tokens spent, append the audit entry.
7. Autonomy: A0 — human decision.
8. Timeout / retry: 5 s / none (the owner can retry from the page).
9. Idempotency: version CAS; a second POST returns the current state.
10. Postconditions: decided approval; tokens spent.
11. Verification: read-back shows the decision, decider and timestamp; audit entry present.
12. Failure routes: invalid or spent token → generic "link no longer valid" page; expired → "expired" page; missing session for high risk → sign-in required.
13. Escalation triggers: more than 5 invalid token attempts per approval → lock that approval's tokens and tell the owner (possible link leak).
14. Evidence: decision, decider principal, channel, timestamp, audit entry ID.

**S6 — Expiry sweep (scheduled)**
1. Objective: no stale approval stays usable.
2. Capability: `nova.approvals.queue.expire_sweep`.
3. Input mapping: now.
4. Dependencies: S3.
5. Preconditions: scheduler running (every 5 minutes).
6. Action / side effects: `pending` past `expires_at` → `expired`; `approved` past `use_by` → `expired`; tokens spent; audit entries appended.
7. Autonomy: A1.
8. Timeout / retry: 30 s / 1 retry.
9. Idempotency: conditional updates; re-running changes nothing.
10. Postconditions: no `pending` or `approved` row past its deadline.
11. Verification: count of overdue rows = 0 after the sweep.
12. Failure routes: sweep fails → `consume` still checks `use_by` and decisions still check `expires_at`, so safety does not depend on the sweep.
13. Escalation triggers: sweep failing for 1 hour → shown on the jobs board.
14. Evidence: rows expired per run.

**S7 — Caller checks status**
1. Objective: let the caller wait for a decision without guessing.
2. Capability: `nova.approvals.queue.status`.
3. Input mapping: `scope-check@0.1.0` (approval ID + full scope).
4. Dependencies: S3.
5. Preconditions: caller is the requester.
6. Action / side effects: read-only; recompute the scope hash and compare.
7. Autonomy: A2.
8. Timeout / retry: 3 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: state, or `scope_mismatch`.
11. Verification: the hash comparison result is returned explicitly.
12. Failure routes: mismatch → the caller must request a new approval.
13. Escalation triggers: none.
14. Evidence: status read time.

**S8 — Consume (single use)**
1. Objective: turn one approval into one permission to act, exactly once.
2. Capability: `nova.approvals.queue.consume`.
3. Input mapping: `scope-check@0.1.0`.
4. Dependencies: S5 (approved).
5. Preconditions: requester identity; recomputed hash equals `scope_hash`; `state = approved`; now < `use_by`.
6. Action / side effects: one conditional update `approved → consumed` with `consumed_at` and a new `grant_id`, plus the audit entry, in one transaction.
7. Autonomy: A1, covered by the owner's decision.
8. Timeout / retry: 5 s / **no retry** — after an ambiguous result, read status.
9. Idempotency: the conditional update affects at most one row.
10. Postconditions: `consume-result` with `granted=true` and `grant_id`, or a refusal reason.
11. Verification: affected row count = 1; audit entry present.
12. Failure routes: `not_approved`, `expired`, `already_consumed`, `scope_mismatch`, `not_requester` → refuse; the caller must not act.
13. Escalation triggers: repeated `scope_mismatch` from one requester → operator notice (possible tampering).
14. Evidence: grant ID, audit entry ID.

**S9 — Record the outcome**
1. Objective: close the loop between approval and reality.
2. Capability: `nova.approvals.queue.record_outcome`.
3. Input mapping: `outcome@0.1.0`.
4. Dependencies: S8.
5. Preconditions: state `consumed`; matching `grant_id`; no outcome yet.
6. Action / side effects: set `outcome` and `outcome_at`; append the audit entry.
7. Autonomy: A1.
8. Timeout / retry: 5 s / 2 retries (idempotent: set once).
9. Idempotency: outcome can be set once; later writes are refused.
10. Postconditions: outcome recorded.
11. Verification: read-back.
12. Failure routes: no outcome within 24 h of consumption → flagged `outcome_missing` on the dashboard.
13. Escalation triggers: `executed_failed` or `outcome_unknown` → shown to the owner with the caller's reference.
14. Evidence: outcome, external reference.

**S10 — Report**
1. Objective: give the caller and the owner a truthful summary.
2. Capability: `nova.approvals.queue.list_pending` (dashboard); run record.
3. Input mapping: approval record and audit entries.
4. Dependencies: the last executed step.
5. Preconditions: none.
6. Action / side effects: read-only; return the final `approval` and the run envelope.
7. Autonomy: A2.
8. Timeout / retry: 3 s / 1 retry.
9. Idempotency: read-only.
10. Postconditions: run result emitted.
11. Verification: every state transition has exactly one audit entry.
12. Failure routes: missing audit entry → integrity alert (should be impossible because of the shared transaction).
13. Escalation triggers: integrity alert → operator.
14. Evidence: approval ID, audit entry IDs.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Input validity | request matches the schema; no secrets in summary or preview | validator, secret scan | valid and clean | reject |
| Human-only decision | `decided_by` is an owner principal; no automation holds `approvals:decide` | grant inventory, record | holds | stop, escalate as a security incident |
| No self-approval | `decided_by` ≠ `requested_by` | record | holds | void the decision, escalate |
| Link safety | GET never changes state | decision endpoint tests, audit | zero state changes on GET | block the endpoint |
| Scope binding | consumed only when the recomputed hash equals `scope_hash` | consume log | 100 % | refuse |
| Single use | at most one `consumed` transition per approval | audit entries | holds | integrity alert |
| Expiry | no consume after `use_by`, no decision after `expires_at` | records | holds | refuse |
| Audit completeness | one audit entry per transition, same transaction | activity log | 1:1 | integrity alert |
| Token hygiene | tokens never stored raw or logged | storage and log scan | none | rotate tokens, purge |
| Output schema | `approval` and `consume-result` valid | validator | valid | fix; never fake |

`granted=true` from `consume` proves permission was used, not that the external action succeeded; S9 records that separately.

## J. Self-correction and recovery

- **Failure classes:** validation, unregistered action, policy refusal, back-pressure, channel failure, invalid token, expired, scope mismatch, ambiguous consume, audit failure.
- **Permitted corrections:** return the existing record for a repeated `request_key`; fall back from chat to dashboard; reissue a prompt with fresh tokens on explicit owner request (old tokens revoked); read status after an ambiguous consume.
- **Forbidden corrections:** extend an expiry; re-open a rejected or expired approval; consume twice; relax the scope match; let a requester decide; skip the audit entry to finish a transition.
- **Maximum correction cycles:** 1 per step.
- **Reverification required after correction:** yes — re-read the record and audit entry.
- **Stop conditions:** hard constraint fails; audit append fails (the transaction rolls back).
- **Recovery or compensation procedure:** the owner revokes an `approved` (unconsumed) approval; after consumption the caller's compensation applies.
- **What must be reported:** fallbacks, refusals with reasons, revocations, outcomes and missing outcomes.

## K. Independent challenge

- **Assumptions to challenge:** the summary truthfully describes the parameters (a caller could write a benign summary over a different payload); the person clicking is the owner; the action is still wanted at consume time.
- **Alternative approach:** for high-impact actions, require the dashboard session even for medium risk if the target is new to the owner.
- **Contradictory evidence:** the preview fields disagree with the summary (e.g. a different recipient) → reject at S1 as `summary_mismatch` when the target display is not mentioned in the summary.
- **Could mislead:** a `consumed` approval read as "sent"; an approval shown without its expiry.
- **Separate verifier:** the scope hash is recomputed by the queue from the parameters the caller presents at consume time — not taken from the caller — so a changed payload cannot ride on an old approval.
- **Findings requiring disclosure:** summary/preview mismatches, revoked approvals, missing outcomes.

## L. Human escalation

- **Triggers:** every approval request (by design); repeated invalid tokens; owner unreachable; `executed_failed` or `outcome_unknown`; integrity alerts.
- **Why the agent cannot decide:** consent to contact a third party belongs to the owner.
- **Information presented:** summary, recipient, preview, risk class, requester, expiry, and for outcomes the caller's reference.
- **Decision requested:** approve or reject this one action; for incidents, lock tokens or disable a requester.
- **Options and consequences:** approve → the caller may act once before `use_by`; reject → the caller must not act; no answer → expiry, no action.
- **Safe behaviour while waiting:** the caller does not act; nothing is consumed.
- **If no response:** the approval expires and the caller is told `expired`.
- **Resume:** a new request with a new approval; the old one cannot be revived.

## M. Completion conditions

Mandatory: approval record valid; every transition audited in the same transaction; decision by an owner principal; scope hash rechecked at consume; single use enforced; expiry enforced; no tokens or secrets stored raw or logged; evidence references present; limitations disclosed.

- `completed` — approved, consumed once and outcome recorded as `executed_ok`; or rejected or cancelled cleanly with the caller informed.
- `partially_completed` — consumed but the outcome is `executed_failed`, `outcome_unknown` or missing; or the prompt fell back to the dashboard.
- `blocked` — rejected at validation or policy, or the owner was unreachable on every channel.
- `failed` — a transaction or audit append failed, or an integrity check failed.
- `cancelled` — the requester cancelled, or the approval expired without a decision or without use.

An approval that expired is never reported as approved, and a consumed approval is never reported as a successful action without an `executed_ok` outcome.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome ("approved by owner via chat-links, consumed once, executed_ok"); final `approval`; `consume-result` where applicable; step statuses S1–S10; verification results and failed checks; evidence references (approval ID, audit entry IDs, message ID, grant ID); approvals and scope (scope hash prefix, action type, expiry); side effects (prompt sent) and reconciliation status (outcome); corrections (fallbacks, reissued prompts); unresolved challenges; limitations; escalation state; learning candidate. Never include tokens, raw parameters or secrets.

## O. Learning and revision

- **Outcome metrics:** approval rate, rejection rate, expiry rate, median time to decision, consume-to-outcome success rate, invalid-token attempts.
- **Evidence retained:** approval records and audit entries for the audit period.
- **Data minimisation:** metrics hold action types and counts only.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}`.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review; changes to decision authentication, single use or expiry need security review and cannot be applied by an automated repair.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (test action type, fictional recipient) | request → prompt → approve → consume → outcome; 5 audit entries | UNVERIFIED on Skill Port; request/prompt/decide/audit observed in the reference implementation |
| Reject path | state `rejected`; consume refused | TBD |
| Missing input (no summary) | rejected at S1 | TBD |
| Invalid input (unregistered action type) | rejected at S2 | TBD |
| Secret in summary | rejected, nothing stored | TBD |
| Missing capability (activity log absent) | activation blocked; no approvals without audit | TBD |
| Permission denied (automation calls `decide`) | denied by grants | TBD |
| Unauthorized scope (another requester consumes) | `not_requester` | TBD |
| Self-approval attempt | denied; security incident raised | TBD |
| Tenant isolation | cannot read or decide another tenant's approvals | TBD |
| Link preview (GET only) | no state change | TBD |
| High risk via chat button | refused; dashboard session required | TBD |
| Timeout on consume | status read; no second consume | TBD |
| Duplicate / replay (same `request_key`; reused token) | existing record returned; spent token refused | TBD |
| Scope change after approval | `scope_mismatch`; new approval needed | TBD |
| Expiry (pending and approved-unused) | `expired`; decisions and consume refused | TBD |
| False success (consume granted, action failed) | outcome `executed_failed`, run `partially_completed` | TBD |
| Failed verification (audit append fails) | transaction rolled back; nothing changed | TBD |
| Correction budget exhausted | stops safely | TBD |
| Escalation (6 invalid tokens) | tokens locked; owner told | TBD |
| Cancellation (requester cancels pending) | `cancelled`; links show invalid | TBD |
| Compensation (owner revokes approved) | `rejected`; consume refused | TBD |
| Output schema and status | valid records; correct status | TBD |
| Compatibility / regression (new action type added) | existing approvals unaffected | TBD |

All tests use test action types and fictional recipients in a sandbox; never a real send. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, the approval UI and whether the platform already provides an approval primitive to reuse instead — TBD (owner: publisher, with Skill Port).
- Audit retention period — TBD (owner: operator, per local requirements).
- Delegated approvers (an assistant or co-founder) are out of scope in 0.1.0; adding them needs per-approver scopes and a new version.
- The summary/preview mismatch rule is a heuristic; it cannot prove a summary is honest. The scope hash binds the parameters, not the wording.
- Payments are deliberately excluded; they need step-up authentication beyond this design.
