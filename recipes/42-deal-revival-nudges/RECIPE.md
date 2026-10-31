# Deal revival nudges

## A. Identity and ownership

- **Recipe ID:** `nova.sales.deal-revival-nudges` (proposed until assigned by Skill Port)
- **Name:** Deal revival nudges
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** marketing
- **Tags:** sales, pipeline, stalled-deals, follow-up, suggestions, owner-digest, no-contact
- **Summary:** Every morning the agent finds leads in the sales pipeline that have gone quiet, drafts a one-line next step and a short suggested opener for each, and sends the owner a single digest. It never contacts a lead; each lead is flagged after the digest is delivered so the owner is not nagged about it every day.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** A sales store with leads, stages, deal values, touches, sequence enrolment state and suppression (reference: PostgreSQL behind security-definer functions); a structured-output LLM; the owner notification channel (reference: a chat bot); a scheduler (reference: n8n, daily at 07:00 in the owner's time zone). All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — reference implementation status: Built. The daily stalled-deal digest with a per-lead opener and the no-repeat flag ran in Nova's reference implementation. The suggestion verifier, cooldown re-eligibility on new activity and send-then-flag ordering are formalised here. Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.sales.pipeline` | Capability | Read the sales pipeline | low | UNVERIFIED |
| `nova.sales.pipeline.stalled` | Operation (read) | Leads in open stages quiet for at least N days, not in an active sequence, not suppressed, not flagged within the cooldown | low | UNVERIFIED |
| `nova.sales.touches` | Capability | Interaction history | low | UNVERIFIED |
| `nova.sales.touches.recent` | Operation (read) | Last touches of a lead as metadata (channel, direction, kind, date, subject line) | low | UNVERIFIED |
| `nova.sales.revival` | Capability | Revival suggestions and the no-repeat flag | low | UNVERIFIED |
| `nova.sales.revival.suggest` | Operation (read, LLM) | Next step and a short opener grounded in the lead's history | low | UNVERIFIED |
| `nova.sales.revival.flag` | Operation (create/update) | Record that a lead was included in a delivered digest | low | UNVERIFIED |
| `nova.notify.owner` | Capability | Deliver a message to the owner's own channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Send the digest | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.sales.revival.suggest.llm` — structured-output LLM, temperature ≤ 0.3, grounded only in lead fields and touch metadata. Selected first.
- `nova.sales.revival.suggest.template` — deterministic fallback with no LLM: next step by stage (for example "engaged, quiet 7 days → send a short check-in referencing your last email of 3 Mar"), no opener text. Eligible when the LLM is unavailable or its output fails verification twice. It uses less data, never more.
- All other operations — only the default variant is supported.

No tool in this recipe can reach a lead: the package binds no email, message or call operation. That absence is part of the design.

**Bindings:** `binding://nova/sales/pipeline-stalled`, `binding://nova/sales/touches-recent`, `binding://nova/sales/revival-suggest`, `binding://nova/sales/revival-flag`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: a scheduled workflow reads a stalled-deals function, calls the LLM, posts the digest to the owner's chat and writes a revival flag table.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Stalled lead (view) | `schema://nova/deal-revival-nudges/stalled-lead@0.1.0` | owner's tenant | read |
| Touch (metadata) | `schema://nova/deal-revival-nudges/touch-summary@0.1.0` | one lead | read |
| Suggestion | `schema://nova/deal-revival-nudges/suggestion@0.1.0` | run-scoped | create (in memory) |
| Revival flag | `schema://nova/deal-revival-nudges/flag-request@0.1.0` | one lead | create, update |

**Authorization requirements (declared, never a grant):** `sales:pipeline:read`, `sales:touches:read`, `sales:revival:write`, `notify:owner`.

**Policy / trust:** risk class `low` — read-only on the pipeline except the flag; the only message goes to the owner.

## C. Intent and applicability

- **User problem:** Deals die quietly because nobody notices that a warm lead has gone silent.
- **Desired outcome:** A short morning list of the stalled leads worth reviving, each with a concrete next step and an opener the owner can adapt, without repeating the same lead every day.
- **Trigger or entry conditions:** Daily schedule (default 07:00 owner time) or the owner asking "which deals are going cold?".
- **Applicable situations:** Leads in open stages (`contacted`, `engaged`, `meeting_booked` whose meeting is in the past without an outcome, `attended`, `proposal`).
- **Out-of-scope situations:** Contacting leads; drafting emails in a mailbox (recipe 10 or an outreach recipe); changing stages; re-enrolling leads into sequences; leads that are won, lost, suppressed or in an active sequence.
- **Required inputs:** Configuration (Section D).
- **Prerequisites:** Touches are logged for every interaction (otherwise "quiet" is meaningless); owner channel connected.
- **Required capabilities:** listed in Section B (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port; every binding is a dependency to resolve before activation.
- **Assumptions:** Verified in the reference implementation: quiet ≥ 5 days and not in an active drip as the stall rule; per-lead opener; flag to avoid daily nags. Assumed: touch logging is complete across channels the owner uses (manual calls may be missing; see Section K).
- **Ambiguity policy:** Use a documented safe default: when the history is too thin to ground an opener, give only the next step and say "not enough history for an opener".
- **Vedaxi task type:** none — not one of the seven task types supported today.

## D. Inputs and outputs

**Inputs (configuration)**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `quiet_days` | integer 2–60 | no | owner | 5 | clamp |
| `cooldown_days` | integer 3–90 | no | owner | 14 | clamp |
| `max_leads` | integer 1–20 | no | owner | 10 | clamp |
| `open_stages` | array of stage names | no | owner | the stages in Section C | unknown stage: reject configuration |
| `time_zone` | IANA zone | yes | owner | — | missing: `blocked` |
| `tone` | enum `warm`, `direct`, `formal` | no | owner | `warm` | unknown: `warm` |

```json
{"$id":"schema://nova/deal-revival-nudges/revival-config@0.1.0","type":"object","required":["time_zone"],"properties":{"quiet_days":{"type":"integer","minimum":2,"maximum":60},"cooldown_days":{"type":"integer","minimum":3,"maximum":90},"max_leads":{"type":"integer","minimum":1,"maximum":20},"open_stages":{"type":"array","items":{"enum":["contacted","engaged","meeting_booked","attended","proposal"]}},"time_zone":{"type":"string"},"tone":{"enum":["warm","direct","formal"]}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| digest message | `digest-message@0.1.0` | `digest_key`, `text`, `lead_ids` | owner's chat | lead names and deal context; owner only | delivery receipt; flagged IDs = listed IDs |
| `run_result` | `run-result@0.1.0` | `run_id`, `status`, `stalled_found`, `listed`, `flagged`, `contacted` | run record | counts | `contacted = 0` always |

```json
{"$id":"schema://nova/deal-revival-nudges/suggestion@0.1.0","type":"object","required":["lead_id","next_step","basis","facts_used"],"properties":{"lead_id":{"type":"string"},"next_step":{"type":"string","minLength":5,"maxLength":160},"opener":{"type":["string","null"],"maxLength":400},"basis":{"enum":["llm","template"]},"facts_used":{"type":"array","items":{"type":"string"},"maxItems":5}}}
```

```json
{"$id":"schema://nova/deal-revival-nudges/run-result@0.1.0","type":"object","required":["run_id","status","stalled_found","listed","flagged","contacted"],"properties":{"run_id":{"type":"string"},"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},"outcome":{"enum":["sent","quiet","not_sent"]},"stalled_found":{"type":"integer","minimum":0},"listed":{"type":"integer","minimum":0},"flagged":{"type":"integer","minimum":0},"template_fallbacks":{"type":"integer","minimum":0},"contacted":{"const":0}}}
```

Other schemas:

```json
{"$id":"schema://nova/deal-revival-nudges/stalled-query@0.1.0","type":"object","required":["quiet_days","cooldown_days","open_stages"],"properties":{"quiet_days":{"type":"integer"},"cooldown_days":{"type":"integer"},"open_stages":{"type":"array","items":{"type":"string"}},"limit":{"type":"integer","minimum":1,"maximum":50}}}
{"$id":"schema://nova/deal-revival-nudges/stalled-lead@0.1.0","type":"object","required":["lead_id","name","stage","days_quiet","last_touch_at"],"properties":{"lead_id":{"type":"string"},"name":{"type":"string"},"company":{"type":["string","null"]},"title":{"type":["string","null"]},"stage":{"type":"string"},"deal_value":{"type":["number","null"]},"currency":{"type":["string","null"]},"days_quiet":{"type":"integer","minimum":0},"last_touch_at":{"type":"string","format":"date-time"},"last_flagged_at":{"type":["string","null"],"format":"date-time"}}}
{"$id":"schema://nova/deal-revival-nudges/touch-query@0.1.0","type":"object","required":["lead_id"],"properties":{"lead_id":{"type":"string"},"limit":{"type":"integer","minimum":1,"maximum":10}}}
{"$id":"schema://nova/deal-revival-nudges/touch-summary@0.1.0","type":"object","required":["lead_id","touches"],"properties":{"lead_id":{"type":"string"},"touches":{"type":"array","maxItems":10,"items":{"type":"object","required":["channel","direction","kind","at"],"properties":{"channel":{"enum":["email","meeting","call","message","other"]},"direction":{"enum":["in","out"]},"kind":{"type":"string"},"subject":{"type":["string","null"],"maxLength":150},"note":{"type":["string","null"],"maxLength":300},"at":{"type":"string","format":"date-time"}}}}}}
{"$id":"schema://nova/deal-revival-nudges/suggest-request@0.1.0","type":"object","required":["lead","touches","tone"],"properties":{"lead":{"$ref":"schema://nova/deal-revival-nudges/stalled-lead@0.1.0"},"touches":{"$ref":"schema://nova/deal-revival-nudges/touch-summary@0.1.0"},"tone":{"enum":["warm","direct","formal"]}}}
{"$id":"schema://nova/deal-revival-nudges/digest-message@0.1.0","type":"object","required":["digest_key","text","lead_ids"],"properties":{"digest_key":{"type":"string"},"text":{"type":"string","maxLength":3500},"lead_ids":{"type":"array","items":{"type":"string"},"maxItems":20}}}
{"$id":"schema://nova/deal-revival-nudges/flag-request@0.1.0","type":"object","required":["lead_ids","digest_key"],"properties":{"lead_ids":{"type":"array","items":{"type":"string"},"minItems":1},"digest_key":{"type":"string"}}}
{"$id":"schema://nova/deal-revival-nudges/flag-result@0.1.0","type":"object","required":["flagged"],"properties":{"flagged":{"type":"integer","minimum":0},"already_flagged":{"type":"integer","minimum":0}}}
{"$id":"schema://nova/deal-revival-nudges/delivery@0.1.0","type":"object","required":["delivered"],"properties":{"delivered":{"type":["boolean","null"]},"message_id":{"type":["string","null"]}}}
```

**Example (fictional)**

Digest text:

```
3 deals are going quiet.

1. Jane Doe, Example Co (proposal, 8 days quiet, 4,000 USD)
   Next: ask whether the proposal from 2 Mar raised questions.
   Opener: "Hi Jane, did the proposal I sent on 2 Mar land well? Happy to walk through any part of it."

2. John Roe, Example Org (engaged, 6 days quiet)
   Next: share the case study he asked about on 28 Feb.
   Opener: not enough history for an opener.
...
Nothing has been sent to anyone.
```

Result: `{"run_id":"run_77","status":"completed","outcome":"sent","stalled_found":5,"listed":3,"flagged":3,"template_fallbacks":0,"contacted":0}`

## E. Constraints and budgets

- **Hard constraints:** never contact a lead (no tool for it is bound); never change stages, sequences or deal values; suggestions cite only facts present in lead fields or touch metadata; no prices, discounts, deadlines or false urgency in openers unless the touch history contains them; flag only leads listed in a delivered digest; skip won, lost, suppressed and actively sequenced leads; one digest per owner per local date.
- **Soft constraints:** order by deal value then days quiet; keep openers under 60 words; the owner's chosen tone.
- **Resource and tenant scope:** the owner's pipeline.
- **Data sources allowed:** leads, touches (metadata, subject lines and owner notes), flags.
- **External transfers permitted:** lead name, company, stage and touch metadata to the LLM provider; the digest to the owner's chat provider.
- **Time limit:** run ≤ 10 minutes.
- **Cost or token budget:** ≤ `max_leads` LLM calls per run; token cap `TBD` (operator).
- **Maximum steps:** 7 (S1–S7); S2–S4 repeat per lead.
- **Maximum retries and correction cycles:** reads 2 retries; LLM 1 regeneration then template fallback; send 2 retries with the same key.
- **Rate limits and concurrency:** single-flight per owner.
- **Retention and logging constraints:** flags kept while the lead is open; logs carry lead IDs and counts.
- **Prohibited actions:** sending, drafting into a mailbox, enrolling, changing stage, deleting anything.
- **Stop conditions:** time zone missing; pipeline store unavailable; owner channel not connected.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | revival flag for a listed lead | digest delivered | Allowed | flag readable |
| Read | leads, touches, flags | own tenant | Allowed | — |
| Update | revival flag `flagged_at`, `digest_key` | re-flag after cooldown | Allowed | re-read |
| Update | lead stage, sequence state, deal fields | — | Not Allowed | — |
| Delete | any | — | Not Allowed | — |
| External side effect | contact with a lead | — | Not Allowed (no binding exists) | `contacted = 0` |
| External side effect | digest to the owner | stalled leads exist | Allowed | delivery receipt |

- **Idempotency:** `digest_key = <owner_id>:<local date>`; flags conditional on the digest key.
- **Expected side effects:** one owner message; flags written.
- **Compensation:** the owner can clear a flag to see the lead again tomorrow.
- **Irreversible:** none beyond the owner having read the message.
- **Ambiguous outcomes:** if the send times out, query the adapter by key; flag only on confirmed delivery; if unknown, do not flag (a repeat tomorrow is better than a silent miss).

## G. Autonomy and approval policy

- **Default autonomy level:** A2 — bounded autonomy (analysis and an owner-only message).
- **Planning level:** A2.
- **Execution level:** A2.
- **Verification level:** A2.
- **Recovery level:** A2.
- **Approval required for:** enabling the recipe; changing the channel. Acting on a suggestion is the owner's own action outside this recipe.
- **Actions never permitted:** contacting leads; drafting in mailboxes; changing pipeline data.
- **Conditions requiring clarification:** none at runtime (configuration only).
- **Conditions requiring escalation:** pipeline store unavailable for 2 consecutive days; touches missing for most leads (logging broken).
- **Safe state after stop, cancellation or escalation:** no flags written without a delivered digest.

## H. Execution plan

**S1 — Find stalled leads**
1. Objective: the leads that genuinely went quiet.
2. Capability: `nova.sales.pipeline.stalled`.
3. Input mapping: `quiet_days`, `cooldown_days`, `open_stages`, limit `max_leads × 2`.
4. Dependencies: none.
5. Preconditions: valid configuration; no confirmed digest today.
6. Action / side effects: read leads in open stages whose last touch (either direction) is at least `quiet_days` old, not in an active sequence, not suppressed, and either never flagged or flagged more than `cooldown_days` ago or with a new touch since the flag.
7. Autonomy: A2; no approval.
8. Timeout / retry: 15 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: stalled leads ordered by deal value, then days quiet.
11. Verification: none is won, lost, suppressed or actively sequenced; `days_quiet ≥ quiet_days`.
12. Failure routes: zero → S5 quiet path; store error → `failed`.
13. Escalation triggers: two consecutive days of store errors.
14. Evidence: lead IDs and days quiet.

**S2 — Gather context (per lead)**
1. Objective: the facts a suggestion may use.
2. Capability: `nova.sales.touches.recent`.
3. Input mapping: `lead_id`, limit 5.
4. Dependencies: S1.
5. Preconditions: lead selected.
6. Action / side effects: read touch metadata (channel, direction, kind, date, subject line, owner note).
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: `touch-summary@0.1.0`.
11. Verification: the newest touch date equals `last_touch_at` from S1.
12. Failure routes: mismatch (new activity since S1) → drop the lead from today's list.
13. Escalation triggers: none.
14. Evidence: touch count.

**S3 — Draft the suggestion (per lead)**
1. Objective: one concrete next step and an optional opener.
2. Capability: `nova.sales.revival.suggest` (variants llm → template).
3. Input mapping: lead, touches, `tone` → `suggest-request@0.1.0`; touch text wrapped as untrusted data.
4. Dependencies: S2.
5. Preconditions: token cap not reached.
6. Action / side effects: one LLM call returning `suggestion@0.1.0` with `facts_used`.
7. Autonomy: A2.
8. Timeout / retry: 60 s / 1 regeneration on S4 failure.
9. Idempotency: not applicable.
10. Postconditions: a candidate suggestion.
11. Verification: S4.
12. Failure routes: LLM error or second verification failure → template variant.
13. Escalation triggers: none.
14. Evidence: basis and token counts.

**S4 — Verify the suggestion (per lead)**
1. Objective: no invented facts, promises or pressure.
2. Capability: local rules; independent check (Section K).
3. Input mapping: suggestion, touch summary, lead.
4. Dependencies: S3.
5. Preconditions: suggestion present.
6. Action / side effects: every item in `facts_used` must appear in the lead fields or touches; reject prices, discounts, deadlines or urgency not in the history; reject openers over 60 words; reject openers that claim actions the owner did not take ("as promised", "following our call" without a call touch).
7. Autonomy: A2.
8. Timeout / retry: 5 s / none.
9. Idempotency: not applicable.
10. Postconditions: approved suggestion or a rejection.
11. Verification: all rules pass.
12. Failure routes: reject → regenerate once (S3) → template fallback.
13. Escalation triggers: none.
14. Evidence: rule results.

**S5 — Compose the digest (or stay quiet)**
1. Objective: one short, readable owner message.
2. Capability: local template.
3. Input mapping: approved suggestions, up to `max_leads`.
4. Dependencies: S4 for all leads, or S1 with zero leads.
5. Preconditions: none.
6. Action / side effects: if no leads, record `quiet` and stop without a message; otherwise list each lead with stage, days quiet, value, next step and opener, ending with "Nothing has been sent to anyone."; plain text ≤ 3,500 characters.
7. Autonomy: A2.
8. Timeout / retry: 2 s / none.
9. Idempotency: deterministic.
10. Postconditions: `digest-message@0.1.0` with listed IDs, or a quiet record.
11. Verification: listed IDs unique; length within limit.
12. Failure routes: too long → reduce `max_leads` and recompose once.
13. Escalation triggers: none.
14. Evidence: listed IDs and content hash.

**S6 — Send to the owner**
1. Objective: deliver the digest once.
2. Capability: `nova.notify.owner.send`.
3. Input mapping: text, `digest_key` as idempotency key.
4. Dependencies: S5.
5. Preconditions: at least one listed lead.
6. Action / side effects: one owner message.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 2 retries with the same key.
9. Idempotency: adapter dedupes on the key.
10. Postconditions: message ID, or unknown.
11. Verification: delivery receipt.
12. Failure routes: failed or unknown → no flags; `failed` or `partially_completed`.
13. Escalation triggers: none.
14. Evidence: message ID.

**S7 — Flag listed leads**
1. Objective: no daily nag about the same lead.
2. Capability: `nova.sales.revival.flag`.
3. Input mapping: listed lead IDs, `digest_key`.
4. Dependencies: S6 delivered.
5. Preconditions: delivery confirmed.
6. Action / side effects: set `flagged_at` and `digest_key` for each listed lead.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 3 retries.
9. Idempotency: conditional on the digest key; repeats report `already_flagged`.
10. Postconditions: `flagged + already_flagged` equals listed.
11. Verification: re-read flags.
12. Failure routes: retries exhausted → `partially_completed`; tomorrow's run may list these leads again (disclosed).
13. Escalation triggers: none.
14. Evidence: `flag-result@0.1.0`.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Configuration | valid `revival-config@0.1.0` | validator | valid | `blocked` |
| Stall rule | each listed lead is quiet ≥ `quiet_days`, open, not sequenced, not suppressed | S1, S2 | holds | drop the lead |
| Grounding | every fact used appears in lead fields or touches | S4 | 100 % | regenerate or template |
| No pressure | no invented price, discount, deadline or urgency | S4 | none | regenerate or template |
| No contact | outbound messages to leads = 0 | outbound logs; absence of bindings | 0 | stop, escalate |
| Send-then-flag | flags only after a delivery receipt | timestamps | flagged_at > sent_at | clear flags, report |
| Listed = flagged | flagged IDs equal listed IDs | S7 | equal | retry; disclose |
| Output schema | `run-result` valid, `contacted = 0` | validator | valid | fix fields |

## J. Self-correction and recovery

- **Failure classes:** configuration, store error, new activity during the run, LLM error, ungrounded suggestion, length overflow, delivery failure, flag failure.
- **Permitted corrections:** drop leads with new activity; regenerate once; fall back to the template; shrink the list; reconcile delivery by key.
- **Forbidden corrections:** inventing context to make an opener sound better; flagging without delivery; contacting a lead "to test interest".
- **Maximum correction cycles:** 1 regeneration per lead; 1 recomposition per run.
- **Reverification required after correction:** yes — S4 on regenerated suggestions; length check on recomposition.
- **Stop conditions:** missing time zone; store unavailable.
- **Recovery or compensation procedure:** the owner clears flags; unflagged leads reappear the next day.
- **What must be reported to the user:** template fallbacks, leads dropped for new activity, delivery or flag problems, and that nothing was sent.

## K. Independent challenge

- **Assumptions to challenge:** that silence in the log means silence in reality (the owner may have phoned or met without logging); that the lead still wants the deal.
- **Alternative approach:** for leads quiet for a very long time, suggest closing them as lost rather than reviving.
- **Contradictory evidence:** an inbound touch on another channel, a calendar booking, or a reply in the mailbox not linked to the lead.
- **Could mislead:** an opener that refers to the wrong last interaction; a deal value that is stale.
- **Separate verifier:** a rule pass, independent of the LLM, checks every date and subject the opener mentions against the touch list.
- **Findings requiring disclosure:** leads with fewer than 2 touches (thin history), stale deal values.

## L. Human escalation

- **Triggers:** pipeline store unavailable for 2 days; touch logging appears broken (most open leads have no touch in 30 days).
- **Why the agent cannot decide:** fixing data capture is an operator or owner task.
- **Information presented:** error class, counts of open leads without touches.
- **Decision requested:** repair logging, pause the recipe or accept reduced quality.
- **Options and consequences:** pausing stops nudges; continuing may produce noisy lists.
- **Safe behaviour while waiting:** no digest, no flags.
- **If no response:** the recipe keeps running daily with the disclosure.
- **Resume:** S1 runs normally on the next schedule.

## M. Completion conditions

Mandatory: run result valid with `contacted = 0`; listed leads satisfy the stall rule; suggestions grounded; flags only for delivered, listed leads; limitations disclosed; evidence references present.

- `completed` — digest delivered and listed leads flagged, or a correct quiet day.
- `partially_completed` — delivery unknown, flags incomplete, or template fallbacks used for more than half the leads.
- `blocked` — configuration missing or owner channel not connected.
- `failed` — the store was unavailable or delivery definitely failed; nothing flagged.
- `cancelled` — the owner paused the recipe before sending.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome ("3 deals flagged for revival; nothing sent to anyone"); `run-result@0.1.0`; step statuses S1–S7; verification results and failed checks; evidence references (lead IDs, digest key, message ID, flag result); approvals and scope; side effects (owner message, flags) and reconciliation status; corrections and retries; unresolved challenges; limitations; escalation state; learning candidate. Never include credentials; lead details only in the owner's channel.

## O. Learning and revision

- **Outcome metrics:** stalled leads per day, leads the owner contacts within 3 days of a nudge (from later touches), revival rate (stage advanced within 14 days), template fallback rate, ungrounded-suggestion rejection rate.
- **Evidence retained:** run results and flags.
- **Data minimisation:** metrics carry counts and lead IDs only.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}`.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (fictional pipeline, 3 quiet leads) | one digest, 3 flags, `contacted = 0` | UNVERIFIED on Skill Port; observed in the reference implementation |
| Quiet day (no stalled leads) | no message | TBD |
| Invalid input (unknown stage in configuration) | `blocked` | TBD |
| Missing input (no time zone) | `blocked` | TBD |
| Missing capability (LLM binding absent) | template suggestions; digest still sent | TBD |
| Permission denied (`sales:revival:write` missing) | digest not sent (flags impossible) and `blocked` | TBD |
| Unauthorized scope (attempt to bind a send tool) | rejected at registration | TBD |
| Tenant isolation | other tenants' leads never listed | TBD |
| Active sequence or suppressed lead | never listed | TBD |
| Cooldown | flagged lead not listed again within 14 days unless new activity | TBD |
| New activity during the run | lead dropped | TBD |
| Timeout on send, adapter confirms | flagged once | TBD |
| Duplicate / replay (run twice in a day) | second run sends nothing | TBD |
| False success (LLM cites a meeting that never happened) | rejected; template used | TBD |
| Failed verification (opener offers a discount) | rejected | TBD |
| Correction budget exhausted | template fallback | TBD |
| Escalation (touch logging broken) | owner told; reduced-quality disclosure | TBD |
| Cancellation (owner pauses) | nothing sent or flagged | TBD |
| Compensation (owner clears a flag) | lead reappears next day if still quiet | TBD |
| Output schema and status | valid `run-result`, `contacted = 0` | TBD |
| Compatibility / regression (new stage added to the pipeline) | ignored unless added to `open_stages` | TBD |

All tests use a fictional pipeline in a sandbox tenant. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs — TBD (owner: publisher, with Skill Port).
- Token cap per run — TBD (owner: operator).
- Accuracy depends on complete touch logging; calls or meetings the owner does not log make leads look quiet.
- Whether to offer a one-tap "draft this opener as an email" hand-off to an approval-gated drafting recipe — open design decision.
- Deal values may be stale; the digest shows them as recorded.
