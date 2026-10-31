# Partner outreach drafts

## A. Identity and ownership

- **Recipe ID:** `nova.sales.partner-outreach-drafts` (proposed until assigned by Skill Port)
- **Name:** Partner outreach drafts
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** marketing
- **Tags:** sales, partnerships, outreach, gmail-draft, never-sends, review-gate, uniqueness-gate, opt-out, follow-ups, reply-drafts
- **Summary:** For contacts the owner has approved, the agent writes short personalised partnership emails as Gmail drafts, never sending them. It keeps to a daily draft budget, rejects drafts that look templated (word-trigram Jaccard similarity above about 0.25), enforces opt-outs everywhere, starts the follow-up clock only when it observes that the owner actually sent the email, and drafts threaded answers when contacts reply.
- **Changelog:** 0.1.0 — Initial draft written as the target design for rebuilding a removed Nova module.
- **Dependencies and compatibility:** A contact store with an owner review gate and suppression (reference: PostgreSQL behind security-definer functions); a mail provider API with draft creation, thread metadata and transient body reads (reference: Gmail API); a structured-output LLM; an approved offer configuration and attachment; the owner notification channel; a scheduler. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Unverified — reference implementation status: Removed (rebuild planned). An earlier version ran in Nova and was removed on the owner's instruction so it could be rebuilt; this recipe is the target design for that rebuild. Design decisions carried over: drafts only, approved contacts only, a shared daily cap with follow-ups first, a uniqueness gate calibrated at about 0.25, a follow-up clock keyed to an observed SENT message, and opt-out suppression. The rebuilt design has not been run, and nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.outreach.queue` | Capability | Work queues and budget for the outreach campaign | medium | UNVERIFIED |
| `nova.outreach.queue.budget` | Operation (read) | Drafts already created today (from the store) and the remaining cap | low | UNVERIFIED |
| `nova.outreach.queue.first_touch` | Operation (read) | Approved, emailable, unsuppressed contacts with no prior draft | low | UNVERIFIED |
| `nova.outreach.queue.follow_up` | Operation (read) | Contacts whose observed SENT message is old enough and who have not replied | low | UNVERIFIED |
| `nova.outreach.queue.replies` | Operation (read) | Detected replies with a non-null thread ID and message ID, not yet handled | low | UNVERIFIED |
| `nova.outreach.compose` | Capability | Write outreach text | medium | UNVERIFIED |
| `nova.outreach.compose.write` | Operation (read, LLM or template) | Subject and personalised lines; code adds offer, CTA, opt-out and signature | medium | UNVERIFIED |
| `nova.outreach.uniqueness` | Capability | Anti-template gate | low | UNVERIFIED |
| `nova.outreach.uniqueness.check` | Operation (read) | Word-trigram Jaccard similarity against the corpus and today's batch | low | UNVERIFIED |
| `nova.outreach.suppression` | Capability | Opt-out and do-not-contact enforcement | high | UNVERIFIED |
| `nova.outreach.suppression.check` | Operation (read) | Is the contact, address or domain suppressed? | low | UNVERIFIED |
| `nova.outreach.suppression.suppress` | Operation (create) | Record an opt-out or bounce and stop all future drafts | medium | UNVERIFIED |
| `nova.mail.observe` | Capability | Observe what happened in a thread at the provider | low | UNVERIFIED |
| `nova.mail.observe.thread_state` | Operation (read) | SENT-labelled messages from the owner and inbound messages after them | low | UNVERIFIED |
| `nova.mail.body` | Capability | Transient body read | high | UNVERIFIED |
| `nova.mail.body.fetch_transient` | Operation (read) | Read one reply body in memory; never persisted | high | UNVERIFIED |
| `nova.outreach.reply` | Capability | Understand replies | medium | UNVERIFIED |
| `nova.outreach.reply.classify` | Operation (read, LLM) | `interested`, `question`, `not_interested`, `opt_out`, `auto_reply`, `bounce` | medium | UNVERIFIED |
| `nova.mail.draft` | Capability | Create drafts in the owner's mailbox | medium | UNVERIFIED |
| `nova.mail.draft.create` | Operation (create at provider) | New-thread or threaded-reply draft; no send | medium | UNVERIFIED |
| `nova.outreach.log` | Capability | Campaign records | low | UNVERIFIED |
| `nova.outreach.log.record` | Operation (create/update) | Record drafted, sent, replied, skipped; add shingles to the corpus | low | UNVERIFIED |
| `nova.notify.owner` | Capability | Deliver a message to the owner's own channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Morning digest | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.outreach.compose.write.first-touch` — LLM writes a subject (≤ 60 characters) and a hook (15–40 words) from the contact's notes; when notes are thin, it uses one of 5 seeded neutral framings and may not assert anything about the contact (claim-word gate).
- `nova.outreach.compose.write.follow-up-1` — LLM adds exactly one new point (≤ 60 words) to the thread.
- `nova.outreach.compose.write.follow-up-2` — deterministic, seeded close-out template; no LLM.
- `nova.outreach.compose.write.reply` — LLM drafts an answer to a classified reply (≤ 120 words), grounded in the reply and the approved offer.
- `nova.mail.draft.create.new-thread` — first touches. `nova.mail.draft.create.threaded-reply` — follow-ups and replies, with thread ID, `In-Reply-To` and `References`. Neither variant can send; there is no fallback between them (a follow-up is never created as a new thread).
- All other operations — only the default variant is supported.

Selection is deterministic by item type (reply → reply; follow-up step 1 → follow-up-1; step 2 → follow-up-2; first touch → first-touch). A failed LLM variant never falls back to sending or to a different contact; it skips the item.

**Bindings:** `binding://nova/outreach/budget`, `binding://nova/outreach/first-touch-queue`, `binding://nova/outreach/follow-up-queue`, `binding://nova/outreach/reply-queue`, `binding://nova/outreach/compose`, `binding://nova/outreach/uniqueness`, `binding://nova/outreach/suppression-check`, `binding://nova/outreach/suppress`, `binding://nova/mail/thread-state`, `binding://nova/mail/body-transient`, `binding://nova/outreach/reply-classify`, `binding://nova/mail/draft-create`, `binding://nova/outreach/log`, `binding://nova/notify/owner` — all `status: UNVERIFIED`.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Outreach contact (approved) | `schema://nova/partner-outreach-drafts/contact@0.1.0` | owner's tenant | read |
| Queue item | `schema://nova/partner-outreach-drafts/queue-item@0.1.0` | one contact + step | read |
| Thread state | `schema://nova/partner-outreach-drafts/thread-state@0.1.0` | one thread at the provider | read |
| Provider draft | `schema://nova/partner-outreach-drafts/draft-result@0.1.0` | owner's sending mailbox | create |
| Touch and shingle record | `schema://nova/partner-outreach-drafts/log-entry@0.1.0` | one contact | create, update |
| Suppression entry | `schema://nova/partner-outreach-drafts/suppression-entry@0.1.0` | owner's tenant | read, create |

**Authorization requirements (declared, never a grant):** `outreach:queue:read`, `outreach:compose`, `outreach:suppression:read`, `outreach:suppression:write`, `mail:thread:read`, `mail:body:read_transient`, `mail:draft:create` (provider compose scope restricted by the adapter to draft create, get and list), `outreach:log:write`, `notify:owner`. Credentials as `secret_ref://sending-mailbox-oauth` and `secret_ref://llm`.

**Policy / trust:** risk class `high` — drafts are addressed to third parties and could be sent with one click; personal data of contacts is processed. Mitigated by the review gate, never-send adapter, opt-out enforcement and the owner pressing send for every email.

## C. Intent and applicability

- **User problem:** Reaching potential partners one by one is slow, and templated mass emails damage reputation and break the law in some places.
- **Desired outcome:** Every working day, up to the cap, ready-to-send personalised drafts for approved contacts, correct follow-ups only after the owner really sent the previous email, threaded reply drafts when people answer, and zero emails to anyone who opted out.
- **Trigger or entry conditions:** Schedule (hourly within business hours, default 09:00–20:00 owner time); a daily digest at 08:00.
- **Applicable situations:** Partnership, affiliate, creator or reseller outreach to contacts the owner has approved.
- **Out-of-scope situations:** Sending; contacts not approved by the owner; importing or sourcing contacts (recipe 41 and the review queue); social-media messaging; bulk newsletters; contacts in opt-in jurisdictions without recorded consent.
- **Required inputs:** Campaign configuration (Section D); approved contacts in the store.
- **Prerequisites:** Contacts imported with warnings (`no_email`, `opt_in_jurisdiction`, `role_address`, `shared_inbox`, `competitor`) and reviewed by the owner (`approved` or `rejected`); an approved offer configuration with figures and an approved attachment; a signature with the sender's postal address and an opt-out line; a seed corpus for the uniqueness gate.
- **Required capabilities:** listed in Section B (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port; every binding is a dependency to resolve before activation.
- **Assumptions:** Carried over from the earlier version (observed then, not in this rebuild): the provider gives a sent message a new ID different from the draft ID, so sent detection matches by thread; word-trigram Jaccard at about 0.25 separated personalised from near-clone drafts on a real corpus where SimHash and embedding similarity did not. Assumed for the rebuild: everything else.
- **Ambiguity policy:** Stop for the item. When contact facts are thin, use a neutral framing and assert nothing about the contact; when a reply's meaning is unclear, classify it `question` and draft a short clarifying answer; when anything about consent or suppression is uncertain, skip the contact and report.
- **Vedaxi task type:** none — not one of the seven task types supported today.

## D. Inputs and outputs

**Inputs (campaign configuration)**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `campaign_id` | string | yes | owner | — | unknown: `blocked` |
| `sending_mailbox_id` | opaque mailbox ID | yes | owner | — | missing: `blocked` |
| `offer` | object with approved figures, text and `approval_hash` | yes | owner-approved | — | hash mismatch: `blocked` |
| `attachment_ref` | approved file reference | no | owner | none | unknown: send without attachment is not allowed; `blocked` |
| `signature` | string containing a postal address and an opt-out line | yes | owner | — | either missing: `blocked` |
| `daily_cap` | integer 1–30 | no | owner | 10 | clamp |
| `reply_cap` | integer 1–50 | no | owner | 20 | clamp |
| `followup_days` | `[d1, d2]` integers | no | owner | `[4, 6]` | invalid: default |
| `similarity_max` | number 0.10–0.40 | no | owner | 0.25 | clamp |
| `dry_run` | boolean | no | owner | false | true: read-only bindings only (Section E) |

```json
{"$id":"schema://nova/partner-outreach-drafts/campaign-config@0.1.0","type":"object","required":["campaign_id","sending_mailbox_id","offer","signature"],"properties":{"campaign_id":{"type":"string"},"sending_mailbox_id":{"type":"string"},"offer":{"type":"object","required":["text","figures","approval_hash"],"properties":{"text":{"type":"string","maxLength":400},"figures":{"type":"object","additionalProperties":{"type":"string"}},"approval_hash":{"type":"string"}}},"attachment_ref":{"type":["string","null"]},"signature":{"type":"string","minLength":20,"maxLength":600},"daily_cap":{"type":"integer","minimum":1,"maximum":30},"reply_cap":{"type":"integer","minimum":1,"maximum":50},"followup_days":{"type":"array","items":{"type":"integer","minimum":1,"maximum":30},"minItems":2,"maxItems":2},"similarity_max":{"type":"number","minimum":0.1,"maximum":0.4},"dry_run":{"type":"boolean"}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| provider drafts | `draft-result@0.1.0` | `draft_id`, `thread_id`, `to`, `is_sent` | owner's mailbox Drafts | personal data | read-back, never sent |
| `run_result` | `run-result@0.1.0` | `run_id`, `status`, `drafted`, `skipped`, `suppressed`, `sent_observed`, `sent_by_agent` | run record; digest | counts and IDs | `sent_by_agent = 0` |

```json
{"$id":"schema://nova/partner-outreach-drafts/run-result@0.1.0","type":"object","required":["run_id","status","drafted","skipped","suppressed","sent_observed","sent_by_agent"],"properties":{"run_id":{"type":"string"},"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},"drafted":{"type":"object","properties":{"first_touch":{"type":"integer"},"follow_up":{"type":"integer"},"reply":{"type":"integer"}}},"skipped":{"type":"object","additionalProperties":{"type":"integer"}},"suppressed":{"type":"integer","minimum":0},"sent_observed":{"type":"integer","minimum":0},"replies_detected":{"type":"integer","minimum":0},"budget_left":{"type":"integer","minimum":0},"dry_run":{"type":"boolean"},"sent_by_agent":{"const":0}}}
```

```json
{"$id":"schema://nova/partner-outreach-drafts/draft-result@0.1.0","type":"object","required":["draft_id","thread_id","to","is_sent"],"properties":{"draft_id":{"type":"string"},"thread_id":{"type":"string"},"to":{"type":"string","format":"email"},"subject":{"type":"string"},"in_reply_to":{"type":["string","null"]},"labels":{"type":"array","items":{"type":"string"}},"is_sent":{"const":false}}}
```

Other schemas:

```json
{"$id":"schema://nova/partner-outreach-drafts/contact@0.1.0","type":"object","required":["contact_id","email","review_status","warnings"],"properties":{"contact_id":{"type":"string"},"name":{"type":["string","null"]},"email":{"type":"string","format":"email"},"notes":{"type":["string","null"],"maxLength":500},"review_status":{"enum":["pending","approved","rejected"]},"warnings":{"type":"array","items":{"enum":["no_email","opt_in_jurisdiction","role_address","shared_inbox","competitor"]}},"consent_recorded":{"type":"boolean"},"inbox_group":{"type":["string","null"]}}}
{"$id":"schema://nova/partner-outreach-drafts/queue-query@0.1.0","type":"object","required":["campaign_id"],"properties":{"campaign_id":{"type":"string"},"limit":{"type":"integer","minimum":1,"maximum":50},"followup_days":{"type":"array","items":{"type":"integer"}}}}
{"$id":"schema://nova/partner-outreach-drafts/budget@0.1.0","type":"object","required":["drafted_today","cap","left","replies_today","reply_left"],"properties":{"drafted_today":{"type":"integer","minimum":0},"cap":{"type":"integer"},"left":{"type":"integer","minimum":0},"replies_today":{"type":"integer","minimum":0},"reply_left":{"type":"integer","minimum":0}}}
{"$id":"schema://nova/partner-outreach-drafts/queue-item@0.1.0","type":"object","required":["contact_id","kind","step"],"properties":{"contact_id":{"type":"string"},"kind":{"enum":["first_touch","follow_up","reply"]},"step":{"type":"integer","minimum":0,"maximum":2},"thread_id":{"type":["string","null"]},"last_message_id":{"type":["string","null"]},"references":{"type":["string","null"]},"sent_observed_at":{"type":["string","null"],"format":"date-time"},"reply_message_ref":{"type":["string","null"]},"contact":{"$ref":"schema://nova/partner-outreach-drafts/contact@0.1.0"}}}
{"$id":"schema://nova/partner-outreach-drafts/compose-request@0.1.0","type":"object","required":["item","variant"],"properties":{"item":{"$ref":"schema://nova/partner-outreach-drafts/queue-item@0.1.0"},"variant":{"enum":["first-touch","follow-up-1","follow-up-2","reply"]},"framing_seed":{"type":"integer","minimum":0,"maximum":4},"reply_text":{"type":["string","null"],"maxLength":20000},"reply_class":{"type":["string","null"]}}}
{"$id":"schema://nova/partner-outreach-drafts/composed@0.1.0","type":"object","required":["subject","personal_text","body_text","facts_poor"],"properties":{"subject":{"type":"string","maxLength":60},"personal_text":{"type":"string","maxLength":800},"body_text":{"type":"string","maxLength":2500},"facts_poor":{"type":"boolean"},"claims":{"type":"array","items":{"type":"string"}}}}
{"$id":"schema://nova/partner-outreach-drafts/uniqueness-request@0.1.0","type":"object","required":["personal_text","campaign_id"],"properties":{"personal_text":{"type":"string"},"campaign_id":{"type":"string"},"batch_texts":{"type":"array","items":{"type":"string"}}}}
{"$id":"schema://nova/partner-outreach-drafts/uniqueness-result@0.1.0","type":"object","required":["max_similarity","passed"],"properties":{"max_similarity":{"type":"number","minimum":0,"maximum":1},"nearest_ref":{"type":["string","null"]},"passed":{"type":"boolean"},"shingle_hashes":{"type":"array","items":{"type":"string"}}}}
{"$id":"schema://nova/partner-outreach-drafts/suppression-query@0.1.0","type":"object","properties":{"contact_id":{"type":"string"},"email":{"type":"string"},"domain":{"type":"string"}},"minProperties":1}
{"$id":"schema://nova/partner-outreach-drafts/suppression-entry@0.1.0","type":"object","required":["suppressed"],"properties":{"suppressed":{"type":"boolean"},"email":{"type":"string"},"reason":{"enum":["opt_out","bounce","complaint","manual","none"]},"since":{"type":["string","null"],"format":"date-time"}}}
{"$id":"schema://nova/partner-outreach-drafts/thread-ref@0.1.0","type":"object","required":["mailbox_id","thread_id"],"properties":{"mailbox_id":{"type":"string"},"thread_id":{"type":"string"},"draft_created_at":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/partner-outreach-drafts/thread-state@0.1.0","type":"object","required":["thread_id","owner_sent","inbound_after_sent"],"properties":{"thread_id":{"type":"string"},"owner_sent":{"type":"array","items":{"type":"object","required":["message_id","sent_at"],"properties":{"message_id":{"type":"string"},"sent_at":{"type":"string","format":"date-time"},"in_reply_to":{"type":["string","null"]}}}},"inbound_after_sent":{"type":"array","items":{"type":"object","required":["message_id","received_at"],"properties":{"message_id":{"type":"string"},"received_at":{"type":"string","format":"date-time"},"delivery_failure":{"type":"boolean"}}}}}}
{"$id":"schema://nova/partner-outreach-drafts/message-ref@0.1.0","type":"object","required":["mailbox_id","message_id"],"properties":{"mailbox_id":{"type":"string"},"message_id":{"type":"string"}}}
{"$id":"schema://nova/partner-outreach-drafts/transient-body@0.1.0","type":"object","required":["text","persist"],"properties":{"text":{"type":"string","maxLength":20000},"persist":{"const":false}}}
{"$id":"schema://nova/partner-outreach-drafts/reply-class@0.1.0","type":"object","required":["class","confidence"],"properties":{"class":{"enum":["interested","question","not_interested","opt_out","auto_reply","bounce"]},"confidence":{"enum":["high","medium","low"]},"auto_reply_until":{"type":["string","null"],"format":"date-time"},"complaint_signal":{"type":"boolean"}}}
{"$id":"schema://nova/partner-outreach-drafts/draft-request@0.1.0","type":"object","required":["mailbox_id","to","subject","body_text","variant"],"properties":{"mailbox_id":{"type":"string"},"to":{"type":"string","format":"email"},"subject":{"type":"string","maxLength":250},"body_text":{"type":"string","maxLength":3000},"variant":{"enum":["new-thread","threaded-reply"]},"thread_id":{"type":["string","null"]},"in_reply_to":{"type":["string","null"]},"references":{"type":["string","null"]},"attachment_ref":{"type":["string","null"]}}}
{"$id":"schema://nova/partner-outreach-drafts/log-entry@0.1.0","type":"object","required":["contact_id","step","kind","state"],"properties":{"contact_id":{"type":"string"},"step":{"type":"integer","minimum":0,"maximum":2},"kind":{"enum":["first_touch","follow_up","reply"]},"state":{"enum":["drafted","sent_observed","replied","skipped","failed"]},"draft_id":{"type":["string","null"]},"thread_id":{"type":["string","null"]},"sent_message_id":{"type":["string","null"]},"sent_observed_at":{"type":["string","null"],"format":"date-time"},"next_due_at":{"type":["string","null"],"format":"date-time"},"skip_reason":{"type":["string","null"]},"shingle_hashes":{"type":"array","items":{"type":"string"}}}}
{"$id":"schema://nova/partner-outreach-drafts/log-receipt@0.1.0","type":"object","required":["entry_id"],"properties":{"entry_id":{"type":"string"},"duplicate":{"type":"boolean"}}}
{"$id":"schema://nova/partner-outreach-drafts/digest-message@0.1.0","type":"object","required":["digest_key","text"],"properties":{"digest_key":{"type":"string"},"text":{"type":"string","maxLength":3500},"item_refs":{"type":"array","items":{"type":"string"}}}}
{"$id":"schema://nova/partner-outreach-drafts/delivery@0.1.0","type":"object","required":["delivered"],"properties":{"delivered":{"type":["boolean","null"]},"message_id":{"type":["string","null"]}}}
```

**Example (fictional)**

Contact: `{"contact_id":"ct_31","name":"Jane Doe","email":"jane@example.com","notes":"Writes a weekly newsletter reviewing workflow tools for small agencies.","review_status":"approved","warnings":[],"consent_recorded":false}`

Draft (first touch, new thread): subject "Your workflow-tools newsletter"; body "Hi Jane, your weekly reviews of workflow tools for small agencies are a great fit for what we build. We run a partner programme with a 15% revenue share for referrals (details in the attached one-pager). Would a short call next week be useful? If you'd rather not hear from us, just reply 'no thanks' and we won't email again. — Sam Lee, Example Co, 100 Example Street, Example City". Similarity to corpus 0.12 → passes. Result: `{"draft_id":"dr_812","thread_id":"thr_2001","to":"jane@example.com","subject":"Your workflow-tools newsletter","in_reply_to":null,"labels":["DRAFT"],"is_sent":false}`.

## E. Constraints and budgets

- **Hard constraints:** never send, schedule-send or forward (draft-only adapter, proven at S1); approved contacts only (`review_status = approved`), never `pending` or `rejected`; never draft for a suppressed contact, address or domain, checked at queue time and again just before draft creation; an opt-out reply suppresses immediately and produces no draft; unsuppressing is an owner action that lands the contact on `paused`, never back in an active state, and no queue operation may reactivate a suppressed contact; contacts flagged `opt_in_jurisdiction` without recorded consent, `no_email` or `competitor` are never drafted; for a `shared_inbox` group only one contact per inbox per campaign; the daily cap counts drafts recorded in the store today (first touches and follow-ups share it, follow-ups first); reply drafts have their own cap; the follow-up clock starts only from an observed SENT message by the owner in the thread, never from draft creation; follow-ups and replies are threaded; offer figures come only from the approved offer configuration — any number written by the LLM is rejected; every draft carries the opt-out line and the signature with a postal address; no claims about the contact beyond the provided notes; personalised text must pass the similarity gate (≤ `similarity_max`, default 0.25); queue items without a thread ID and message ID are never processed as replies; each item fails on its own; in `dry_run` mode the run uses read-only bindings only and writes nothing anywhere (no suppress, no log, no draft).
- **Soft constraints:** 90–130 words per first touch; plain, friendly tone; the contact's first name when known, otherwise "Hi,".
- **Resource and tenant scope:** the owner's campaign, contacts and sending mailbox.
- **Data sources allowed:** approved contacts and notes, campaign configuration, thread metadata, transient reply bodies, the shingle corpus.
- **External transfers permitted:** contact notes, reply text and draft text to the LLM provider; drafts to the owner's mail provider.
- **Time limit:** run ≤ 15 minutes.
- **Cost or token budget:** LLM calls ≤ (cap + reply cap) × 2 per day; token cap `TBD` (operator).
- **Maximum steps:** 11 (S1–S11).
- **Maximum retries and correction cycles:** provider reads 2 retries; draft create 1 attempt plus reconciliation; LLM 1 regeneration (a different seeded framing for the similarity gate).
- **Rate limits and concurrency:** single-flight per campaign; hourly runs; provider quotas.
- **Retention and logging constraints:** shingle corpus keeps hashed trigrams, not raw text; touches kept per sales-data retention (`TBD`); logs carry IDs, not addresses or bodies.
- **Prohibited actions:** sending; contacting rejected or pending contacts; reactivating suppressed contacts; writing offer figures via the LLM; attaching any file other than the approved attachment; deleting or editing the owner's own drafts.
- **Stop conditions:** draft-only restriction unproven; offer hash mismatch; signature lacks a postal address or opt-out line; kill switch on.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | provider draft (new thread or threaded reply) | approved, unsuppressed contact; gates passed; within cap | Allowed (pre-approved at campaign activation: drafts only) | read-back unsent |
| Create | suppression entry (opt-out, bounce) | classified reply or bounce | Allowed | entry readable; contact leaves all queues |
| Create | touch records and hashed shingles | after each draft, observed send or reply | Allowed | unique per contact + step |
| Read | contacts, queues, budget, thread metadata, transient reply body | own tenant, sending mailbox | Allowed | — |
| Update | touch state (`drafted` → `sent_observed` → `replied`), follow-up due date | observation-based | Allowed | re-read |
| Update | `review_status`, unsuppress, offer, signature | — | Not Allowed (owner actions) | — |
| Delete | any draft, contact or record | — | Not Allowed | — |
| External side effect | sending email | — | Not Allowed (adapter refuses) | `sent_by_agent = 0` |
| External side effect | owner digest | daily | Allowed | delivery receipt |

- **Idempotency:** one draft per (contact, step) enforced by a unique key; before a second create, check the thread (or the mailbox for first touches by recorded key) for an existing draft; suppression keyed by address.
- **Expected side effects:** drafts in the owner's mailbox; suppression entries; LLM tokens.
- **Compensation:** the owner deletes unwanted drafts; the recipe never deletes; wrong suppressions are reversed by the owner (to `paused`).
- **Irreversible:** content sent to the LLM provider.
- **Ambiguous outcomes:** on a create timeout, look for the draft by thread and creation time before anything else; never create two drafts for one step.

## G. Autonomy and approval policy

- **Default autonomy level:** A1 — supervised execution: drafts are created within a narrow pre-approved scope; the owner sends every email.
- **Planning level:** A2.
- **Execution level:** A1 for draft creation; A2 for observation, suppression and records.
- **Verification level:** A2.
- **Recovery level:** A2 — skip and report; never widen scope.
- **Approval required for:** activating a campaign (scope: drafts only, approved contacts, this offer hash, this attachment, this cap); approving each contact (the review gate); changing the offer, signature, attachment or cap; unsuppressing a contact.
- **Actions never permitted:** sending; drafting for unapproved or suppressed contacts; reactivating suppressed contacts; LLM-written figures; attachments other than the approved one.
- **Conditions requiring clarification:** a reply that both asks a question and says stop (treated as opt-out; the owner is told).
- **Conditions requiring escalation:** draft-only restriction unproven; a SENT label on a draft the owner did not send (should be impossible); bounce rate on observed sends above 5 %; any complaint signal.
- **Safe state after stop, cancellation or escalation:** no new drafts; existing drafts untouched; suppressions kept.

## H. Execution plan

**S1 — Preflight**
1. Objective: prove the run is safe before touching any contact.
2. Capability: installation binding check (UNVERIFIED); configuration validation.
3. Input mapping: `campaign-config@0.1.0`.
4. Dependencies: none.
5. Preconditions: kill switch off.
6. Action / side effects: verify the adapter exposes no send operation; offer `approval_hash` matches; signature contains a postal address and an opt-out line; attachment approved; in `dry_run`, swap to read-only bindings; acquire the single-flight lock.
7. Autonomy: A2; no approval.
8. Timeout / retry: 10 s / 1 retry.
9. Idempotency: read-only.
10. Postconditions: preflight passed.
11. Verification: every check recorded pass/fail.
12. Failure routes: any failure → `blocked` with the reason.
13. Escalation triggers: send operation reachable → escalate.
14. Evidence: preflight result.

**S2 — Observe sends and replies**
1. Objective: learn what the owner actually sent and who answered.
2. Capability: `nova.mail.observe.thread_state`; `nova.outreach.log.record`; `nova.outreach.suppression.suppress` (bounces).
3. Input mapping: every touch in state `drafted` or `sent_observed` with a thread ID.
4. Dependencies: S1.
5. Preconditions: preflight passed.
6. Action / side effects: for each thread, find a SENT message from the owner created after the draft (matched by thread, then `In-Reply-To` or subject, because the sent ID differs from the draft ID) → record `sent_observed` with its time and message ID and set the next follow-up due date; find inbound messages after the owner's last sent → enqueue a reply with thread ID and message ID; delivery-failure notices → suppress as `bounce`.
7. Autonomy: A2.
8. Timeout / retry: 15 s per thread / 2 retries.
9. Idempotency: state transitions are conditional; a re-observation changes nothing.
10. Postconditions: touches updated; replies enqueued.
11. Verification: every `sent_observed` touch has a provider message ID and a SENT label; every enqueued reply has a thread ID and message ID.
12. Failure routes: a thread unreadable → skip it this run; never infer "sent" from the draft disappearing alone.
13. Escalation triggers: SENT on a message the owner did not send → stop and escalate.
14. Evidence: counts of observed sends and replies.

**S3 — Handle replies**
1. Objective: honour opt-outs instantly and draft helpful answers.
2. Capability: `nova.outreach.queue.replies`; `nova.mail.body.fetch_transient`; `nova.outreach.reply.classify`; `nova.outreach.suppression.suppress`; then S8–S10 for drafts.
3. Input mapping: reply items (contact, thread ID, message ID).
4. Dependencies: S2.
5. Preconditions: item has a non-null thread ID and message ID; reply cap not reached.
6. Action / side effects: read the reply body in memory; classify; `opt_out` → suppress, mark the contact stopped, no draft; `bounce` → suppress; `auto_reply` → ignore and keep the follow-up schedule paused for its stated period (default 7 days); `interested`, `question`, `not_interested` → compose with the reply variant (a `not_interested` answer is a brief courteous close with no persuasion) and continue at S8.
7. Autonomy: A2 (suppression); A1 (draft).
8. Timeout / retry: 20 s read, 60 s classify / 1 retry each.
9. Idempotency: one handling per reply message ID.
10. Postconditions: each reply is suppressed, ignored or has a draft queued.
11. Verification: opt-outs produce no draft; the contact disappears from every queue.
12. Failure routes: body unreadable or classifier fails → skip this item, keep it queued with backoff; never default to drafting.
13. Escalation triggers: complaint language (for example "spam", "report") → suppress and tell the owner.
14. Evidence: classification and action per reply (no body text).

**S4 — Compute today's budget**
1. Objective: stay within the daily cap whatever happened earlier today.
2. Capability: `nova.outreach.queue.budget`.
3. Input mapping: campaign, owner local date.
4. Dependencies: S3.
5. Preconditions: none.
6. Action / side effects: read drafts recorded today from the store (not from memory or logs).
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: `budget@0.1.0`.
11. Verification: `left = cap − drafted_today` (never negative).
12. Failure routes: budget unreadable → `blocked` for new drafts (replies already queued still wait).
13. Escalation triggers: none.
14. Evidence: budget snapshot.

**S5 — Follow-ups first**
1. Objective: continue started conversations before opening new ones.
2. Capability: `nova.outreach.queue.follow_up`; `nova.outreach.compose.write` (follow-up-1 or follow-up-2).
3. Input mapping: `followup_days`, budget left.
4. Dependencies: S4.
5. Preconditions: budget left > 0.
6. Action / side effects: select contacts whose step-0 send was observed at least d1 days ago (step 1) or whose step-1 send was observed at least d2 days ago (step 2), with no reply, not suppressed, not paused; compose; continue at S8 per item.
7. Autonomy: A2.
8. Timeout / retry: 60 s per compose / 1 regeneration.
9. Idempotency: unique (contact, step).
10. Postconditions: follow-up drafts queued for S8, up to the budget.
11. Verification: every selected item has `sent_observed_at`; no item has a later inbound message.
12. Failure routes: compose fails twice → skip the item for today.
13. Escalation triggers: none.
14. Evidence: selected items and step numbers.

**S6 — Select first touches**
1. Objective: pick the next approved contacts within the remaining budget.
2. Capability: `nova.outreach.queue.first_touch`; `nova.outreach.suppression.check`.
3. Input mapping: campaign, remaining budget.
4. Dependencies: S5.
5. Preconditions: budget left after follow-ups > 0.
6. Action / side effects: read contacts with `review_status = approved`, an email, no blocking warnings (`no_email`, `competitor`, `opt_in_jurisdiction` without consent), not suppressed, replied or booked, no prior draft, and at most one per `inbox_group`.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: candidate contacts.
11. Verification: every candidate is approved and passes the suppression check.
12. Failure routes: suppression check unavailable → no first touches this run.
13. Escalation triggers: none.
14. Evidence: contact IDs.

**S7 — Compose first touches**
1. Objective: short, honest, personal openers.
2. Capability: `nova.outreach.compose.write.first-touch`.
3. Input mapping: contact name and notes, a seeded framing (0–4) when notes are thin.
4. Dependencies: S6.
5. Preconditions: token cap not reached.
6. Action / side effects: the LLM writes subject and hook only; code assembles the body: greeting, hook, the approved offer sentence with figures from configuration, "details in the attached one-pager" when an attachment exists, one call to action, the opt-out line and the signature.
7. Autonomy: A2.
8. Timeout / retry: 60 s / 1 regeneration.
9. Idempotency: not applicable.
10. Postconditions: `composed@0.1.0`.
11. Verification: no digits or currency in LLM-written text; when `facts_poor`, no claim words about the contact (for example "loved your post", "your audience"); subject ≤ 60 characters; total 90–130 words.
12. Failure routes: rule failure → regenerate once → else skip.
13. Escalation triggers: none.
14. Evidence: rule results and framing seed.

**S8 — Uniqueness gate (every draft)**
1. Objective: no draft that looks like a template.
2. Capability: `nova.outreach.uniqueness.check`.
3. Input mapping: personalised text (subject, hook and any LLM lines; fixed approved blocks excluded), today's batch.
4. Dependencies: S3, S5 or S7.
5. Preconditions: composed text present.
6. Action / side effects: lower-case, strip punctuation, build word trigrams, compute Jaccard similarity against the campaign corpus and today's batch; take the maximum.
7. Autonomy: A2.
8. Timeout / retry: 5 s / none.
9. Idempotency: deterministic.
10. Postconditions: `uniqueness-result@0.1.0`.
11. Verification: pass when the maximum ≤ `similarity_max`.
12. Failure routes: above the threshold → regenerate once with a different framing seed → still above → skip the contact for today with reason `near_clone` (follow-up-2 templates and reply drafts are compared only within their own variant).
13. Escalation triggers: more than half of today's candidates skipped as near-clones → tell the owner the notes are too thin.
14. Evidence: maximum similarity and the nearest corpus item ID.

**S9 — Create the draft**
1. Objective: a correct draft in the owner's mailbox; never a send.
2. Capability: `nova.mail.draft.create` (new-thread or threaded-reply).
3. Input mapping: to = contact email; subject; body; approved attachment; for threaded: thread ID, `In-Reply-To` = last message ID, `References`.
4. Dependencies: S8.
5. Preconditions: suppression re-checked within the last 5 minutes; budget still available; not `dry_run`.
6. Action / side effects: one draft created.
7. Autonomy: A1 under the campaign activation approval.
8. Timeout / retry: 20 s / no blind retry — reconcile first.
9. Idempotency: unique (contact, step); look for an existing draft before any second create.
10. Postconditions: `draft-result@0.1.0`.
11. Verification: read-back: label `DRAFT`, not `SENT`; recipient equals the contact; threaded drafts carry the thread ID and headers.
12. Failure routes: provider error → skip this item (the next item continues); read-back mismatch → record `failed` and tell the owner (no deletion).
13. Escalation triggers: `SENT` on read-back → stop and escalate.
14. Evidence: draft ID and read-back.

**S10 — Record the draft**
1. Objective: the store, the budget and the corpus reflect reality.
2. Capability: `nova.outreach.log.record`.
3. Input mapping: contact, step, kind, draft ID, thread ID, hashed shingles.
4. Dependencies: S9.
5. Preconditions: draft created.
6. Action / side effects: record touch `drafted`; add hashed shingles to the corpus.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 3 retries.
9. Idempotency: unique index on (contact, step) blocks duplicates.
10. Postconditions: `log-entry@0.1.0` stored.
11. Verification: re-read; the budget count includes this draft.
12. Failure routes: write fails → stop creating new drafts this run (the budget would be wrong) and report `partially_completed`; the next run's S2 finds the draft in the thread and records it.
13. Escalation triggers: none.
14. Evidence: log entry ID.

**S11 — Morning digest (separate daily run)**
1. Objective: the owner knows what is waiting.
2. Capability: `nova.notify.owner.send`.
3. Input mapping: drafts awaiting send, replies drafted, sends observed yesterday, new suppressions, near-clone skips, budget.
4. Dependencies: the previous day's runs.
5. Preconditions: at least one item to report.
6. Action / side effects: one plain-text owner message at 08:00; quiet if nothing; items marked reported only after delivery (send-then-mark).
7. Autonomy: A2.
8. Timeout / retry: 15 s / 2 retries with the same key.
9. Idempotency: key = campaign + local date.
10. Postconditions: delivered, or quiet.
11. Verification: delivery receipt; reported items = listed items.
12. Failure routes: undelivered → items stay unreported for tomorrow.
13. Escalation triggers: none.
14. Evidence: message ID.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Never send | adapter has no send; read-backs lack `SENT` | S1, S9 | holds | stop, escalate |
| Review gate | every drafted contact is `approved` | S6 vs store | 100 % | stop drafting; report |
| Suppression | no draft for a suppressed contact; checked at S6 and S9 | suppression store | none | stop; report |
| Opt-out | every opt-out reply suppressed with no draft | S3 | 100 % | suppress; escalate |
| Budget | drafts today ≤ cap; replies ≤ reply cap | store counts | within | stop new drafts |
| Follow-up timing | follow-up only after an observed SENT ≥ d days ago | touches | holds | skip |
| Figures | no number written by the LLM; figures equal configuration | S7 | holds | regenerate or skip |
| Uniqueness | max Jaccard ≤ threshold | S8 | holds | regenerate or skip |
| Reply integrity | reply items have thread ID and message ID | S3 | holds | skip item |
| Dry run | zero writes in `dry_run` | write audit | 0 | stop; report a defect |
| Output schema | `run-result` valid, `sent_by_agent = 0` | validator | valid | fix fields |

A draft existing proves nothing about sending; only an observed SENT message counts as sent.

## J. Self-correction and recovery

- **Failure classes:** preflight failure, provider read error, malformed queue item, classifier failure, compose rule failure, near-clone, provider create error, ambiguous create, record failure, budget unreadable.
- **Permitted corrections:** skip an item and continue; regenerate once with a new framing seed; reconcile an ambiguous create by looking in the thread; back off a failing reply item.
- **Forbidden corrections:** sending; lowering the similarity threshold to fill the budget; inventing facts to personalise; drafting for unapproved or suppressed contacts; treating draft disappearance as a send.
- **Maximum correction cycles:** 1 regeneration per item; 1 reconciliation per create.
- **Reverification required after correction:** yes — S7 rules and S8 gate after regeneration; read-back after any create.
- **Stop conditions:** preflight failure; `SENT` on read-back; record failure (budget integrity).
- **Recovery or compensation procedure:** the owner deletes unwanted drafts and reverses wrong suppressions to `paused`.
- **What must be reported to the user:** drafts created by type, skips with reasons, suppressions, observed sends, near-clone rate, and that nothing was sent by the agent.

## K. Independent challenge

- **Assumptions to challenge:** that an approved contact still wants partnership emails; that the notes are accurate; that a SENT message in the thread is the owner's send of our draft and not an unrelated email.
- **Alternative approach:** for high-value partners, a personal email written by the owner may outperform; offer "notes only" mode that drafts talking points instead of an email.
- **Contradictory evidence:** replies saying "wrong person", bounces, unusually low reply rates for one framing seed.
- **Could mislead:** a fluent hook that implies familiarity the owner does not have; a digest that counts drafts as outreach done.
- **Separate verifier:** a rule pass independent of the composer checks figures, claim words, opt-out line, postal address and recipient; the similarity gate is computed independently of the LLM.
- **Findings requiring disclosure:** thin-notes contacts, framing seeds converging (rising similarity), suppressions.

## L. Human escalation

- **Triggers:** draft-only restriction unproven; unexpected `SENT`; complaint language; bounce rate above 5 %; near-clone skips above half; offer or signature invalid.
- **Why the agent cannot decide:** reputation, legal compliance and consent are the owner's responsibility.
- **Information presented:** counts, affected contact IDs, sample (owner-only) of the problem text, the failing check.
- **Decision requested:** fix the binding or configuration, pause the campaign, enrich notes, or accept the skips.
- **Options and consequences:** pausing stops drafts; observation and suppression continue.
- **Safe behaviour while waiting:** no new drafts; opt-outs still honoured.
- **If no response:** the campaign stays paused for that condition; the digest repeats the request weekly.
- **Resume:** S1 re-runs every preflight check.

## M. Completion conditions

Mandatory: run result valid with `sent_by_agent = 0`; every draft for an approved, unsuppressed contact, within the cap, passed the gates and read-back; every opt-out suppressed with no draft; follow-ups only after observed sends; dry runs wrote nothing; limitations disclosed; evidence references present.

- `completed` — observation done, replies handled, follow-ups and first touches drafted up to the budget or until the queues were empty.
- `partially_completed` — some items skipped (near-clone, compose or provider errors), a record write failed, or the digest was not delivered.
- `blocked` — preflight failed (send reachable, offer hash, signature, attachment, configuration) or the budget or suppression check was unavailable.
- `failed` — provider or store unavailable for the whole run; nothing drafted.
- `cancelled` — the owner paused the campaign or switched the kill switch on mid-run.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome ("4 follow-ups, 6 first touches, 2 reply drafts ready; 1 opt-out suppressed; 0 sent by the agent"); `run-result@0.1.0`; step statuses S1–S11; verification results and failed checks; evidence references (draft IDs, read-backs, suppression IDs, similarity scores); approvals and scope (campaign activation, offer hash); side effects (drafts, suppressions) and reconciliation status; corrections and retries; unresolved challenges; limitations; escalation state; learning candidate. Never include bodies, credentials or full addresses in shared logs.

## O. Learning and revision

- **Outcome metrics:** drafts per day by type, owner send rate (observed), reply rate by framing seed, opt-out and bounce rates, near-clone skip rate, median similarity, time from draft to observed send.
- **Evidence retained:** touches, suppression entries, hashed shingle corpus.
- **Data minimisation:** metrics without addresses or text; corpus stores hashed trigrams only.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}`.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests; similarity-threshold changes must be re-calibrated on a labelled corpus.
- **Promotion approval:** publisher review plus Skill Port review; any change to sending capability, review gate or suppression needs explicit sign-off.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (sandbox mailbox, 3 fictional approved contacts) | 3 unsent drafts, logged, digest next morning | UNVERIFIED — design target; the earlier removed version was observed, the rebuild has not run |
| Invalid input (signature without postal address) | `blocked` | TBD |
| Missing input (no offer approval hash) | `blocked` | TBD |
| Missing capability (draft binding absent) | `blocked` | TBD |
| Permission denied (`mail:draft:create` missing) | `blocked` | TBD |
| Unauthorized scope (adapter exposes send) | `blocked`, escalated | TBD |
| Tenant isolation | another tenant's contacts never queued | TBD |
| Pending or rejected contact | never drafted | TBD |
| Suppressed contact; unsuppress | never drafted; unsuppress lands on `paused` | TBD |
| Opt-out reply | suppressed, no draft, leaves all queues | TBD |
| Follow-up without observed send | not drafted | TBD |
| Observed send with a new message ID | matched by thread; follow-up scheduled | TBD |
| Daily cap with follow-ups due | follow-ups first; total ≤ cap | TBD |
| Near-clone (similarity 0.31) | regenerated, then skipped | TBD |
| LLM writes a figure | rejected | TBD |
| Reply item with null thread ID | skipped; run continues | TBD |
| Timeout on create, draft exists | adopted; no duplicate | TBD |
| Duplicate / replay (run twice in an hour) | no second draft per step | TBD |
| False success (create OK, read-back shows wrong recipient) | `failed`, owner told | TBD |
| Dry run | zero writes anywhere | TBD |
| Correction budget exhausted | item skipped with reason | TBD |
| Escalation (complaint reply) | suppressed, owner told | TBD |
| Cancellation (kill switch mid-run) | stops after the current item | TBD |
| Compensation (owner deletes a draft) | recipe does not recreate it | TBD |
| Output schema and status | valid `run-result`, `sent_by_agent = 0` | TBD |
| Compatibility / regression (provider changes sent-message IDs) | thread matching still detects sends | TBD |

All tests use a sandbox mailbox and fictional contacts at example.com; never real people. Not production-ready until the mandatory tests have evidence and the rebuild has run in a sandbox.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, and whether the platform can restrict a mail grant to draft operations — TBD (owner: publisher, with Skill Port).
- Similarity threshold calibration for the rebuild's corpus and languages — TBD (owner: publisher); 0.25 is a starting point carried over, not a verified value for new data.
- Opt-in jurisdiction list and consent recording — operator configuration; this recipe gives no legal advice.
- Whether `not_interested` replies should get any draft at all — open design decision (current design: a brief courteous close).
- Reply-draft cap and follow-up spacing defaults — TBD (owner: operator).
- The owner's real postal address must be supplied before activation where local email law requires it.
