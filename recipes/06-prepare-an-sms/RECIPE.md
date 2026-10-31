# Prepare an SMS

## A. Identity and ownership

- **Recipe ID:** `nova.comms.prepare-an-sms` (proposed until assigned by Skill Port)
- **Name:** Prepare an SMS
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** productivity
- **Tags:** sms, text-message, drafting, opt-out, quiet-hours, sender-id, human-approval · Vedaxi task type: Prepare an SMS
- **Summary:** On an explicit request, the agent resolves the recipient, drafts one short SMS, measures its length and encoding, checks opt-out status, quiet hours and the destination country's sender rules, and asks the user to approve the exact text. By default the approved text is handed to the user to send from their own phone; sending through an SMS provider is possible only when a provider binding is installed (UNVERIFIED — not built in the reference implementation) and always needs per-message approval.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** An LLM for drafting; a contact store; an owner notification channel. For the provider-send variant only: an SMS provider account with a sender registered for the destination country, an opt-out register and an inbound-reply handler. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Unverified — partly built. Nova's reference implementation can draft messages and has inbound SMS capture (messages forwarded from the user's phone into a webhook gateway), but it has **no outbound SMS sender and no opt-out register**. The provider-send variant is a design only. Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.contacts.directory` | Capability | Find a person's phone number from approved sources | medium | UNVERIFIED |
| `nova.contacts.directory.lookup` | Operation (read) | Look a name up in assistant memory, then the contact directory | medium | UNVERIFIED |
| `nova.comms.sms.compose` | Capability | Draft one SMS and measure it | low | UNVERIFIED |
| `nova.comms.sms.compose.draft` | Operation (read, LLM) | Write the message text from the user's intent | low | UNVERIFIED |
| `nova.comms.sms.compose.measure` | Operation (read; rules) | Return encoding (GSM-7 or UCS-2), character count and segment count | low | UNVERIFIED |
| `nova.comms.sms.policy` | Capability | Decide whether this SMS may be sent, now, to this number | high | UNVERIFIED |
| `nova.comms.sms.policy.check` | Operation (read) | Country allowlist, sender-ID rules, quiet hours, content category, caps | high | UNVERIFIED |
| `nova.comms.sms.optout` | Capability | Register of numbers that opted out | high | UNVERIFIED — not built |
| `nova.comms.sms.optout.lookup` | Operation (read) | Is this number opted out for this sender? | high | UNVERIFIED — not built |
| `nova.comms.sms.optout.record` | Operation (create; used by the inbound-reply handler, not by this run) | Record STOP and similar replies | high | UNVERIFIED — not built |
| `nova.comms.sms.send` | Capability | Deliver one approved SMS | high | UNVERIFIED |
| `nova.comms.sms.send.message` | Operation (external side effect, or human hand-off) | Send through a provider, or hand the text to the user | high | UNVERIFIED |
| `nova.comms.sms.status` | Capability | Delivery state of a provider-sent SMS | low | UNVERIFIED — not built |
| `nova.comms.sms.status.get` | Operation (read) | Return `accepted`, `queued`, `sent`, `delivered`, `undelivered` or `failed` | low | UNVERIFIED — not built |
| `nova.notify.owner` | Capability | Deliver a message to the requesting user's own channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Show the draft, collect approval, report the result | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.contacts.directory.lookup.memory` → `.directory` → `.ask-user` — as in the prepare-a-call recipe: read-only memory first, read-only directory second, then ask the user once. Never guess a number.
- `nova.comms.sms.send.message.handoff` — **default.** Human-assisted: the agent returns the approved text and an `sms:` link (RFC 5724) that opens the user's own messaging app with the recipient and body filled in; the user presses send. The agent sends nothing and cannot confirm delivery. Binding `binding://nova/comms/sms-handoff`. Built as far as drafting and returning text; the link format is a proposal.
- `nova.comms.sms.send.message.provider-api` — an SMS provider's REST API with a sender registered for the destination country. Binding `binding://nova/comms/sms-provider` (UNVERIFIED, **not built**). Eligible only when the binding is installed and healthy, the opt-out register is installed, the inbound-reply handler is installed, the user holds `sms:send`, and the user chose agent-send for this message.
- `nova.comms.sms.compose.draft.default`, `nova.comms.sms.compose.measure.default`, `nova.comms.sms.policy.check.default`, `nova.comms.sms.optout.lookup.default`, `nova.comms.sms.optout.record.default`, `nova.comms.sms.status.get.default`, `nova.notify.owner.send.default` — only the default variant is supported.

**Variant selection policy:** `handoff` unless every `provider-api` eligibility condition holds and the user chose agent-send at the approval step. The two send variants are never silent fallbacks for each other: if an approved provider send becomes impossible (binding down, opt-out register unavailable), the agent stops and asks whether to hand off instead; it never sends through a different provider or number without a new approval.

**Bindings:** `binding://nova/contacts/lookup`, `binding://nova/comms/sms-compose`, `binding://nova/comms/sms-measure`, `binding://nova/comms/sms-policy`, `binding://nova/comms/sms-optout-lookup`, `binding://nova/comms/sms-optout-record`, `binding://nova/comms/sms-handoff`, `binding://nova/comms/sms-provider`, `binding://nova/comms/sms-status`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: drafting runs in the assistant; inbound SMS capture exists as an authenticated webhook gateway that receives messages forwarded from the user's phone; no outbound sender, opt-out register or delivery-receipt handler exists.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Contact | `schema://nova/prepare-an-sms/contact@0.1.0` | the requesting user | read |
| SMS draft | `schema://nova/prepare-an-sms/sms-draft@0.1.0` | this run | create, update (edits before approval) |
| Opt-out record | `schema://nova/prepare-an-sms/optout-record@0.1.0` | per sender and recipient number | read (this run); create (inbound handler only) |
| Outbound SMS | `schema://nova/prepare-an-sms/sms-record@0.1.0` | the requesting user's tenant | create (provider variant only), read |

**Authorization requirements (declared, never a grant):** `contacts:read`, `sms:optout:read`, `sms:send` (provider variant only, approval required per message), `sms:status:read`, `notify:owner`; `sms:optout:write` belongs to the inbound-reply handler. Provider credentials live in the runtime as `secret_ref://sms-provider`; LLM key `secret_ref://llm`.

**Policy / trust:** risk class `high` for the provider variant (a message reaches a third party under a business sender, with legal rules on consent, opt-out and timing); `medium` for hand-off (the user sends personally). Requires per-message approval, the opt-out check, quiet hours, a country allowlist and sender-ID compliance.

## C. Intent and applicability

- **User problem:** The user wants a short text message written and checked — clear, the right length, appropriate for the recipient — without composing it themselves.
- **Desired outcome:** One approved SMS, either handed to the user ready to send or (when a provider is installed) sent and tracked to a truthful delivery state.
- **Trigger or entry conditions:** The user explicitly asks to text or SMS a named person or number and says what the message is about.
- **Applicable situations:** Confirming a time, a short reminder, "running late", passing on a simple message, a one-to-one transactional notice the user personally requests.
- **Out-of-scope situations:** Bulk, marketing or promotional SMS; one-time passcodes; messages to emergency or short-code numbers; messages that contain passwords, full account or card numbers, or health details; impersonating anyone; messages to numbers outside the allowlist; scheduling SMS for later (not in this version).
- **Required inputs:** `recipient`, `intent` (see Section D).
- **Prerequisites:** A contact source; an LLM. Provider variant only: a provider account, a sender registered for the destination country, an opt-out register and an inbound-reply handler.
- **Required capabilities:** `nova.contacts.directory`, `nova.comms.sms.compose`, `nova.comms.sms.policy`, `nova.comms.sms.send`, `nova.notify.owner`; provider variant also `nova.comms.sms.optout`, `nova.comms.sms.status` (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** Outbound SMS provider, opt-out register, inbound-reply handler and delivery-receipt handling are **not built** in the reference implementation. Until they exist, only the hand-off variant can run.
- **Assumptions:** Verified in the reference implementation: drafting and inbound SMS capture. Assumed: the target runtime can resolve the recipient's country and time zone from the number or the contact record.
- **Ambiguity policy:** Ask. If the recipient or the content is unclear, ask one question; never guess a number or invent content.
- **Vedaxi task type:** Prepare an SMS.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `recipient` | string: a name, or an E.164 number (`^\+[1-9][0-9]{7,14}$`) | yes | user; personal data | — | Ask once. Invalid number: reject and ask. Two directory matches: ask which. |
| `intent` | string, 5–500 chars | yes | user; may be personal | — | Ask for it; never send without content from the user. |
| `exact_text` | string, ≤ 918 chars | no | user | — | When given, used verbatim (only measured and checked, not rewritten). |
| `delivery_mode` | enum `handoff`, `agent_send` | no | user | `handoff` | `agent_send` with no eligible provider: explain and offer hand-off. |
| `max_segments` | integer 1–6 | no | runtime policy | 2 | Out of range: clamp and tell the user. |
| `language` | BCP 47 tag | no | user or inferred | the user's language | Non-Latin scripts force UCS-2 (70 chars per single segment); disclosed in the preview. |

```json
{"$id":"schema://nova/prepare-an-sms/request@0.1.0","type":"object","required":["recipient","intent"],
 "properties":{"recipient":{"type":"string","minLength":2},"intent":{"type":"string","minLength":5,"maxLength":500},
 "exact_text":{"type":"string","maxLength":918},"delivery_mode":{"enum":["handoff","agent_send"]},
 "max_segments":{"type":"integer","minimum":1,"maximum":6},"language":{"type":"string"}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `sms_result` | object `schema://nova/prepare-an-sms/sms-result@0.1.0` | `status`, `text`, `encoding`, `segments`, `delivery_mode`, `delivery_state`, `to_masked` | the requesting user's channel and the run record | personal data; owner only | approval hash, policy log and (provider) delivery receipt (Section I) |

```json
{"$id":"schema://nova/prepare-an-sms/sms-result@0.1.0","type":"object",
 "required":["status","text","encoding","segments","delivery_mode","delivery_state","to_masked"],
 "properties":{"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},
 "text":{"type":"string","maxLength":918},"encoding":{"enum":["GSM-7","UCS-2"]},
 "chars":{"type":"integer","minimum":0},"segments":{"type":"integer","minimum":1,"maximum":6},
 "delivery_mode":{"enum":["handoff","provider_api","none"]},
 "delivery_state":{"enum":["not_sent","handed_off_unconfirmed","accepted","queued","sent","delivered","undelivered","failed"]},
 "to_masked":{"type":"string"},"handoff_link":{"type":["string","null"]},
 "provider_message_id":{"type":["string","null"]},"approval_id":{"type":["string","null"]},
 "cost_estimate":{"type":["object","null"],"properties":{"amount":{"type":"number"},"currency":{"type":"string"}}},
 "checks":{"type":"array","items":{"type":"object","required":["name","result"],
   "properties":{"name":{"type":"string"},"result":{"enum":["pass","fail","not_applicable"]},"reason":{"type":"string"}}}}}}
```

Helper schemas (draft, version 0.1.0; each `$id` matches the manifest reference):

```json
{"$id":"schema://nova/prepare-an-sms/contact-query@0.1.0","type":"object","required":["query"],"properties":{"query":{"type":"string","minLength":2},"max_results":{"type":"integer","minimum":1,"maximum":5}}}
{"$id":"schema://nova/prepare-an-sms/contact@0.1.0","type":"object","required":["name","phone_e164","source"],"properties":{"name":{"type":"string"},"phone_e164":{"type":"string","pattern":"^\\+[1-9][0-9]{7,14}$"},"country":{"type":"string","pattern":"^[A-Z]{2}$"},"time_zone":{"type":"string"},"source":{"enum":["memory","directory","user"]}}}
{"$id":"schema://nova/prepare-an-sms/sms-draft@0.1.0","type":"object","required":["text","encoding","chars","segments","text_sha256"],"properties":{"text":{"type":"string","minLength":1,"maxLength":918},"encoding":{"enum":["GSM-7","UCS-2"]},"chars":{"type":"integer","minimum":1},"segments":{"type":"integer","minimum":1,"maximum":6},"text_sha256":{"type":"string"}}}
{"$id":"schema://nova/prepare-an-sms/policy-input@0.1.0","type":"object","required":["phone_e164","country","time_zone","delivery_mode","check_at"],"properties":{"phone_e164":{"type":"string","pattern":"^\\+[1-9][0-9]{7,14}$"},"country":{"type":"string","pattern":"^[A-Z]{2}$"},"time_zone":{"type":"string"},"delivery_mode":{"enum":["handoff","agent_send"]},"content_category":{"enum":["personal","transactional"]},"segments":{"type":"integer","minimum":1,"maximum":6},"check_at":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/prepare-an-sms/policy-result@0.1.0","type":"object","required":["checks","eligible_modes"],"properties":{"checks":{"type":"array","items":{"type":"object","required":["name","result"],"properties":{"name":{"type":"string"},"result":{"enum":["pass","fail","not_applicable"]},"reason":{"type":"string"}}}},"eligible_modes":{"type":"array","items":{"enum":["handoff","provider_api"]}},"next_window_at":{"type":["string","null"],"format":"date-time"},"cost_estimate":{"type":["object","null"],"properties":{"amount":{"type":"number"},"currency":{"type":"string"}}}}}
{"$id":"schema://nova/prepare-an-sms/optout-query@0.1.0","type":"object","required":["sender_ref","phone_e164"],"properties":{"sender_ref":{"type":"string"},"phone_e164":{"type":"string","pattern":"^\\+[1-9][0-9]{7,14}$"}}}
{"$id":"schema://nova/prepare-an-sms/optout-record@0.1.0","type":"object","required":["sender_ref","phone_e164","opted_out"],"properties":{"sender_ref":{"type":"string"},"phone_e164":{"type":"string","pattern":"^\\+[1-9][0-9]{7,14}$"},"opted_out":{"type":"boolean"},"opted_out_at":{"type":["string","null"],"format":"date-time"},"keyword":{"type":["string","null"]},"source":{"enum":["reply","user","operator",null]}}}
{"$id":"schema://nova/prepare-an-sms/send-input@0.1.0","type":"object","required":["variant","to_e164","text","text_sha256","approval_id","idempotency_key"],"properties":{"variant":{"enum":["handoff","provider-api"]},"to_e164":{"type":"string","pattern":"^\\+[1-9][0-9]{7,14}$"},"text":{"type":"string","minLength":1,"maxLength":918},"text_sha256":{"type":"string"},"approval_id":{"type":"string"},"idempotency_key":{"type":"string"},"sender_ref":{"type":["string","null"]}}}
{"$id":"schema://nova/prepare-an-sms/sms-record@0.1.0","type":"object","required":["message_id","variant","to_e164","text_sha256","approval_id","state","state_at"],"properties":{"message_id":{"type":"string"},"provider_message_id":{"type":["string","null"]},"variant":{"enum":["handoff","provider-api"]},"to_e164":{"type":"string","pattern":"^\\+[1-9][0-9]{7,14}$"},"sender_ref":{"type":["string","null"]},"text_sha256":{"type":"string"},"approval_id":{"type":"string"},"state":{"enum":["handed_off_unconfirmed","accepted","queued","sent","delivered","undelivered","failed"]},"state_at":{"type":"string","format":"date-time"},"cost":{"type":["object","null"],"properties":{"amount":{"type":"number"},"currency":{"type":"string"}}}}}
{"$id":"schema://nova/prepare-an-sms/sms-ref@0.1.0","type":"object","anyOf":[{"required":["message_id"]},{"required":["provider_message_id"]},{"required":["idempotency_key"]}],"properties":{"message_id":{"type":"string"},"provider_message_id":{"type":"string"},"idempotency_key":{"type":"string"}}}
{"$id":"schema://nova/prepare-an-sms/delivery@0.1.0","type":"object","required":["message_id","status"],"properties":{"message_id":{"type":"string"},"status":{"enum":["delivered","failed"]},"delivered_at":{"type":["string","null"],"format":"date-time"},"channel":{"type":"string"}}}
```

**Segment rules used by `measure`:** GSM-7 — 160 characters in one segment, 153 per segment when split; characters from the GSM-7 extension table (for example `€ [ ] { } ~ ^ | \`) count as two. UCS-2 — 70 characters in one segment, 67 per segment when split; any character outside GSM-7 (most emoji, non-Latin scripts) switches the whole message to UCS-2.

**Example (fictional)**

Input: `{"recipient":"Jane Doe","intent":"Tell her I'm 10 minutes late for our 3 pm and she can start without me."}`

Output: `{"status":"completed","text":"Hi Jane, it's Alex. Running about 10 minutes late for our 3 pm. Please start without me. Sorry!","encoding":"GSM-7","chars":95,"segments":1,"delivery_mode":"handoff","delivery_state":"handed_off_unconfirmed","to_masked":"…0142","handoff_link":"sms:+15550142?body=Hi%20Jane%2C%20it%27s%20Alex.%20Running%20about%2010%20minutes%20late%20for%20our%203%20pm.%20Please%20start%20without%20me.%20Sorry%21","provider_message_id":null,"approval_id":"apr_0091","cost_estimate":null,"checks":[{"name":"opt_out","result":"not_applicable","reason":"no opt-out register installed; personal hand-off"},{"name":"allowlist","result":"pass"},{"name":"segments","result":"pass"}]}`

## E. Constraints and budgets

- **Hard constraints:** explicit user request; per-message approval of the exact text (hash-bound) before any send or hand-off; content only from the user's intent — no invented facts, promises or links; never send to a number on the opt-out register; the provider variant cannot run without an installed opt-out register and inbound-reply handler; destination country on the allowlist; quiet hours for the provider variant (default 09:00–20:00 in the recipient's time zone; stricter local rules win); sender must be registered as the destination country requires (for example: US application-to-person traffic on 10-digit long codes needs brand and campaign registration, toll-free numbers need verification, and alphanumeric sender IDs are not supported in the US or Canada; India requires registration of sender headers and templates; many countries require pre-registered alphanumeric sender IDs — confirm with the provider's current country guidelines, UNVERIFIED); never include passwords, one-time codes, full account or card numbers, or health details; never impersonate — provider-sent messages identify the sender; no marketing or bulk messages; one message per approval.
- **Soft constraints:** one segment when possible; plain words, no URL shorteners (carriers often filter them); the user's usual tone and language.
- **Resource and tenant scope:** the requesting user's contacts and outbound SMS records only.
- **Data sources allowed:** assistant memory, the user's contact directory, the user's own message.
- **External transfers permitted:** the intent to the configured LLM; for the provider variant, the number and text to the SMS provider solely to deliver this message.
- **Time limit:** drafting ≤ 60 s; approval wait ≤ 30 minutes; delivery-receipt wait ≤ 10 minutes (provider variant).
- **Cost or token budget:** per-message cap `TBD` (set by the operator); the estimate = segments × the provider's per-segment price for the destination; stop if it exceeds the cap.
- **Maximum steps:** 9 (S1–S9).
- **Maximum retries and correction cycles:** provider send: 0 automatic re-sends; reads: 2 retries; redrafting: 1 automatic plus up to 3 user-requested edits; correction cycles: 1 per step, 3 per run.
- **Rate limits and concurrency:** provider variant: at most 1 message per recipient per 10 minutes and at most 20 messages per user per day (TBD); provider rate limits apply.
- **Retention and logging constraints:** outbound records keep the text hash, state and cost; message text retained per the operator's policy (`TBD`); logs never contain credentials or full numbers in shared channels.
- **Prohibited actions:** re-sending without a new approval; changing the text after approval; sending from an unregistered sender; sending to an opted-out number; bypassing quiet hours.
- **Stop conditions:** approval declined or timed out; opt-out on file; policy check fails; cost over cap; provider variant ineligible and the user declines hand-off.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | SMS draft (in-run) | request valid | Allowed | measure and S5 checks pass |
| Update | SMS draft text (before approval only) | user asked for an edit | Allowed | re-measured and re-checked |
| Create | Outbound SMS record (provider variant, one per approval) | S1–S6 passed; all provider eligibility met | Approval Required (S6) | record with the provider message ID |
| Read | Contact, opt-out record, outbound SMS state | own tenant; opt-out per sender | Allowed | — |
| Delete | any | — | Not Allowed | — |
| External side effect | SMS to the recipient (provider variant) | approved hash, allowlist, opt-out clear, quiet hours, registered sender, cost cap | Approval Required (S6) | delivery receipt `delivered` |
| External side effect | hand-off text and link to the requesting user | approved hash | Approval Required (S6) | the user receives the hand-off message |
| External side effect | result message to the requesting user | run finished | Allowed | delivery receipt |

- **Idempotency:** provider send key = hash(`run_id`, `to_e164`, `text_sha256`, `approval_id`); a repeated send with the same key returns the existing record instead of sending again.
- **Expected side effects:** provider variant — the recipient's phone receives the message and charges accrue; hand-off — none until the user sends.
- **Compensation:** none — a delivered SMS cannot be recalled. If a wrong message was sent, the agent can draft a correction for a new approval.
- **Irreversible:** a sent SMS and its charges.
- **Ambiguous outcomes:** if the provider call times out, read status by idempotency key (S8) before anything else; never re-send on an unknown outcome.

## G. Autonomy and approval policy

- **Default autonomy level:** A0 — assisted (the user sends, via hand-off). A1 for the provider variant when installed.
- **Planning level:** A2 (resolve, draft, measure, check).
- **Execution level:** A0 for hand-off; A1 for provider send, only after the S6 approval.
- **Verification level:** A2.
- **Recovery level:** A1 — no re-send without a new approval.
- **Approval required for:** the exact text and recipient (every message); the delivery mode; a cost estimate over the default.
- **Actions never permitted:** sending without approval; bulk or marketing SMS; sending to opted-out numbers; one-time codes or secrets in messages; impersonation; using an unregistered sender; sending during quiet hours.
- **Conditions requiring clarification:** unknown or ambiguous recipient; missing intent; text over `max_segments`.
- **Conditions requiring escalation:** the recipient is opted out but the user insists; country not allowlisted; repeated provider failures; cost over cap.
- **Safe state after stop, cancellation or escalation:** nothing sent; the draft kept in the run record; the user told why.

Approval covers one message with the shown text hash, recipient and delivery mode. It does not cover re-sends, edits after approval or other recipients.

## H. Execution plan

**S1 — Validate the request**
1. Objective: confirm recipient and intent are present and valid.
2. Capability: none (local validation).
3. Input mapping: user message → `request@0.1.0`.
4. Dependencies: none.
5. Preconditions: explicit request to text someone.
6. Action / side effects: parse and validate; no side effects.
7. Autonomy: A2; no approval.
8. Timeout / retry: 5 s / none.
9. Idempotency: not applicable.
10. Postconditions: schema-valid request.
11. Verification: JSON Schema validation passes.
12. Failure routes: missing or invalid field → ask once → else `blocked`.
13. Escalation triggers: the request is bulk or marketing → explain it is out of scope.
14. Evidence: validated request.

**S2 — Resolve the recipient**
1. Objective: exactly one E.164 number with country and time zone.
2. Capability: `nova.contacts.directory.lookup` (memory → directory → ask-user).
3. Input mapping: `recipient` → `query`.
4. Dependencies: S1.
5. Preconditions: `recipient` is a name (skip lookup if already a number).
6. Action / side effects: read-only lookups; asking the user is a message to the user only.
7. Autonomy: A2 for reads; A0 for ask-user.
8. Timeout / retry: 10 s per variant / 2 retries on transient errors.
9. Idempotency: read-only.
10. Postconditions: one `contact@0.1.0`.
11. Verification: E.164 pattern; exactly one match; country derived from the number.
12. Failure routes: several matches → ask which; none → ask-user; user declines → `blocked`.
13. Escalation triggers: none.
14. Evidence: which source supplied the number.

**S3 — Policy pre-check**
1. Objective: know early whether the message can go out and by which mode.
2. Capability: `nova.comms.sms.policy.check`; `nova.comms.sms.optout.lookup` when installed.
3. Input mapping: `phone_e164`, country, time zone, `delivery_mode`.
4. Dependencies: S2.
5. Preconditions: number resolved.
6. Action / side effects: check allowlist, opt-out status, content category (personal or transactional only), provider eligibility (binding, sender registered for the country, opt-out register and inbound handler present), per-recipient and daily caps; quiet hours are evaluated now and again at S7; no side effects.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 1 retry.
9. Idempotency: read-only.
10. Postconditions: a check list with pass, fail or not_applicable and reasons; the eligible delivery modes.
11. Verification: every hard check has a recorded result.
12. Failure routes: opted out → `blocked` ("this number opted out of messages from this sender"); not allowlisted → `blocked`; `agent_send` requested but provider ineligible → offer hand-off.
13. Escalation triggers: the user insists on messaging an opted-out number → explain and stop.
14. Evidence: check results.

**S4 — Draft the message**
1. Objective: a short, clear SMS that says only what the user wants.
2. Capability: `nova.comms.sms.compose.draft` (skipped when `exact_text` is given).
3. Input mapping: `intent`, recipient first name, the user's display name, `language`, `max_segments`.
4. Dependencies: S3.
5. Preconditions: at least one delivery mode eligible.
6. Action / side effects: write the text; for provider sends, identify the sender and, where the jurisdiction or carrier requires it, add opt-out wording ("Reply STOP to opt out"); no side effects.
7. Autonomy: A2.
8. Timeout / retry: 30 s / 1 retry.
9. Idempotency: not applicable.
10. Postconditions: an `sms-draft@0.1.0`.
11. Verification: see S5.
12. Failure routes: drafting fails → ask the user for the exact text.
13. Escalation triggers: none.
14. Evidence: draft text and hash.

**S5 — Measure and verify the draft**
1. Objective: the draft fits and contains nothing it should not.
2. Capability: `nova.comms.sms.compose.measure` plus rule checks.
3. Input mapping: draft text.
4. Dependencies: S4.
5. Preconditions: draft exists.
6. Action / side effects: compute encoding, characters and segments; check for secrets, one-time-code patterns, full account or card numbers, URL shorteners and facts not in the intent; no side effects.
7. Autonomy: A2.
8. Timeout / retry: 5 s / none.
9. Idempotency: not applicable.
10. Postconditions: measured draft with check results.
11. Verification: segments ≤ `max_segments`; no forbidden content; no added facts.
12. Failure routes: too long → shorten once → still too long → show the user the count and ask; forbidden content → remove and tell the user.
13. Escalation triggers: none.
14. Evidence: encoding, segments, check results.

**S6 — Approval checkpoint**
1. Objective: explicit approval of this exact message and mode.
2. Capability: `nova.notify.owner.send` (user interaction).
3. Input mapping: recipient name, masked number (last 4 digits), text, encoding and segments, eligible modes, cost estimate (provider), any check notes.
4. Dependencies: S5.
5. Preconditions: draft verified.
6. Action / side effects: ask "Text Jane Doe (…0142) this message?" with Approve and hand off / Approve and send (only if provider-eligible) / Edit / Cancel.
7. Autonomy: A0 — human decision.
8. Timeout / retry: wait up to 30 minutes, then `cancelled`.
9. Idempotency: the approval ID binds one text hash, one recipient and one mode.
10. Postconditions: an approval record with ID, hash, mode and timestamp.
11. Verification: the approval references the current draft hash.
12. Failure routes: Edit → back to S4/S5 (up to 3 rounds); Cancel or timeout → `cancelled`.
13. Escalation triggers: cost over cap → explicit confirmation of the higher amount.
14. Evidence: approval ID, hash, mode, timestamp.

**S7 — Deliver (hand off or send)**
1. Objective: carry out the approved delivery mode exactly once.
2. Capability: `nova.comms.sms.send.message` (`handoff` or `provider-api`).
3. Input mapping: `phone_e164`, approved text, approval ID, idempotency key.
4. Dependencies: S6.
5. Preconditions: approval valid; text hash unchanged; for provider: opt-out re-checked, quiet hours and caps re-checked, binding healthy.
6. Action / side effects: hand-off — send the user the text and an `sms:` link; provider — submit one message (the recipient's phone receives it; charges accrue).
7. Autonomy: A0 (hand-off) / A1 (provider), covered by the S6 approval.
8. Timeout / retry: 20 s for the provider call; **no automatic re-send**.
9. Idempotency: provider key = hash(`run_id`, `to_e164`, `text_sha256`, `approval_id`).
10. Postconditions: hand-off delivered to the user, or a provider record with a provider message ID in `accepted` or `queued`.
11. Verification: the record's `to` and text hash equal the approved values.
12. Failure routes: provider timeout → S8 status read by key; provider rejection → `failed` with the reason; re-check fails (now in quiet hours, opted out) → stop, tell the user, offer hand-off.
13. Escalation triggers: repeated provider errors → tell the user.
14. Evidence: hand-off message ID, or provider message ID and timestamps.

**S8 — Confirm delivery state**
1. Objective: a truthful final state.
2. Capability: `nova.comms.sms.status.get` (provider); none for hand-off.
3. Input mapping: provider message ID or idempotency key.
4. Dependencies: S7.
5. Preconditions: provider variant used; skipped for hand-off (state stays `handed_off_unconfirmed` unless the user says it was sent).
6. Action / side effects: read delivery receipts; `accepted`, `queued` and `sent` are never reported as delivered.
7. Autonomy: A2.
8. Timeout / retry: poll or receive receipts for up to 10 minutes / 2 read retries.
9. Idempotency: read-only.
10. Postconditions: final state `delivered`, `undelivered` or `failed`, or `sent` with no receipt by the deadline.
11. Verification: state comes from the provider's receipt, not from the submit response.
12. Failure routes: no receipt by the deadline → report `sent` (unconfirmed) → `partially_completed`.
13. Escalation triggers: `undelivered` → tell the user and offer another channel.
14. Evidence: state transitions with timestamps.

**S9 — Report**
1. Objective: tell the user what happened.
2. Capability: `nova.notify.owner.send`.
3. Input mapping: `sms_result`.
4. Dependencies: S8 (or S7 for hand-off).
5. Preconditions: result assembled.
6. Action / side effects: one message to the requesting user.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 2 retries.
9. Idempotency: message key = `run_id`.
10. Postconditions: delivered.
11. Verification: delivery receipt.
12. Failure routes: delivery fails → result kept in the run record → `partially_completed`.
13. Escalation triggers: none.
14. Evidence: message ID.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Input validity | request matches `request@0.1.0` | validator | valid | ask once, then `blocked` |
| Number identity | sent or handed-off number = approved number | record vs approval | exact match | stop; report |
| Text identity | sent text hash = approved hash | record vs approval | equal | stop before sending |
| Opt-out | number not opted out (provider: register required) | opt-out lookup at S3 and S7 | clear | `blocked` |
| Policy | allowlist, sender registration, quiet hours, caps | policy log at S3 and S7 | all pass | `blocked` or hand-off offer |
| Content | no secrets, codes, full account numbers, added facts | S5 rules | none | remove; re-approve |
| Length | segments ≤ `max_segments`; encoding correct | measure | within | shorten or ask |
| Delivery truth | `delivered` only from a provider receipt; hand-off never reported as delivered | S8 | consistent | correct the result |
| Output schema | `sms_result` valid | validator | valid | fix fields; never fake values |

A provider accepting a message proves only that it was queued, not that it reached the phone.

## J. Self-correction and recovery

- **Failure classes:** validation, ambiguous recipient, opt-out, policy block, provider ineligible, length overflow, forbidden content, timeout, provider rejection, missing receipt, permission denial.
- **Permitted corrections:** re-ask once; shorten once; remove forbidden content and re-approve; reconcile an ambiguous send by reading status; offer hand-off when the provider is ineligible (needs the user's choice).
- **Forbidden corrections:** re-send without approval; edit approved text; switch sender, provider or number silently; ignore an opt-out; report hand-off as delivered.
- **Maximum correction cycles:** 1 per step; 3 per run.
- **Reverification required after correction:** yes — S5 and S6 after any text change; S3 checks again at S7.
- **Stop conditions:** correction budget exhausted; hard constraint fails.
- **Recovery or compensation:** none possible for a delivered SMS; offer a correction message for a new approval.
- **What must be reported:** mode used, final state, charges, any check that failed, any change to the draft.

## K. Independent challenge

- **Assumptions to challenge:** the number belongs to the intended person; the recipient expects messages from this sender; the message reads as intended in a short form.
- **Alternative approach:** a WhatsApp message or a call may suit better (longer content, need for a reply); offer when the text exceeds two segments or delivery failed.
- **Contradictory evidence:** an opt-out on file, a recent `undelivered` state for the number, a contact note saying "prefers email".
- **Could mislead:** reporting `sent` as delivered; a draft that implies a commitment the user did not make; missing sender identification on a business message.
- **Separate verifier:** S5 rule checks run independently of drafting; delivery state comes only from provider receipts.
- **Findings requiring disclosure:** UCS-2 switch and extra segments, opt-out notes, unconfirmed hand-off, missing receipts.

## L. Human escalation

- **Triggers:** ambiguous recipient; opted-out recipient; country not allowlisted; cost over cap; repeated provider failure; message over the segment limit.
- **Why the agent cannot decide:** consent, cost and wording that speaks for the user are the user's decisions; an opt-out is a legal line the agent cannot cross.
- **Information presented:** recipient (masked), text, segments, checks with reasons, cost estimate, delivery state.
- **Decision requested:** approve / edit / cancel; hand off instead; accept a longer message.
- **Options and consequences:** approving a provider send delivers a message that cannot be recalled and accrues charges; hand-off leaves sending to the user.
- **Safe behaviour while waiting:** nothing sent.
- **If no response:** after 30 minutes the run ends `cancelled`.
- **Resume:** a new approval is required; S3 checks re-run first.

## M. Completion conditions

Mandatory: `sms_result` valid; approval hash equals the delivered or handed-off text; opt-out and policy checks recorded; delivery state truthful; charges reported (provider); limitations disclosed; evidence references present.

- `completed` — hand-off: the approved text and link were delivered to the user (state `handed_off_unconfirmed`, reported as such); provider: the receipt shows `delivered`.
- `partially_completed` — provider: `sent` but no final receipt by the deadline; or the message was delivered but the report to the user failed.
- `blocked` — invalid input, opt-out, allowlist, sender registration, quiet hours, cost cap, or agent-send requested without an eligible provider and hand-off declined.
- `failed` — the provider rejected the message or the receipt shows `undelivered` or `failed`.
- `cancelled` — the user cancelled or approval timed out.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome; `sms_result`; step statuses S1–S9; verification results and failed checks; evidence references (approval ID, text hash, provider message ID or hand-off message ID, receipt timestamps); approvals and scope; side effects (message sent or handed off, charges) and reconciliation status; corrections performed; unresolved challenges; limitations (provider variant not built, unconfirmed hand-off); escalation state; learning candidate. Never include credentials or full phone numbers in shared channels.

## O. Learning and revision

- **Outcome metrics:** approval rate, edit rounds, average segments, UCS-2 rate, hand-off vs provider share, delivered rate, opt-out blocks, cost per message.
- **Evidence retained:** run record, approval record and delivery states, per the retention policy.
- **Data minimisation:** metrics stored without message text or numbers.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}`.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review; no silent changes to trusted behaviour; the provider variant cannot be activated by a revision without its opt-out tests passing.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (hand-off) | measured, approved text and link handed to the user | UNVERIFIED on Skill Port; drafting observed in the reference implementation |
| Happy path (provider sandbox) | sent once; receipt `delivered` | TBD — provider variant not built |
| Missing intent | asks once, then `blocked` | TBD |
| Invalid number | rejected before policy checks | TBD |
| Ambiguous recipient | asks which person | TBD |
| Length overflow (UCS-2 text over 2 segments) | shortened or user asked | TBD |
| Forbidden content (one-time code in text) | removed; re-approval required | TBD |
| Missing capability (no provider, `agent_send` requested) | explains; offers hand-off | TBD |
| Missing opt-out register with provider installed | provider variant ineligible | TBD |
| Opted-out recipient | `blocked`; nothing sent | TBD |
| Quiet hours (provider) | `blocked` with the next window, or hand-off offered | TBD |
| Permission denied (`sms:send` missing) | provider ineligible; hand-off only | TBD |
| Unauthorized scope (another tenant's contact) | denied | TBD |
| Tenant isolation | cannot read another tenant's SMS records or opt-outs | TBD |
| Timeout on provider send | status read by key; no second send | TBD |
| Retry on transient read error | bounded retries | TBD |
| Duplicate / replay | same key returns the existing record | TBD |
| False success (provider accepted, receipt `undelivered`) | `failed`, not delivered | TBD |
| Failed verification (draft adds a fact) | redrafted | TBD |
| Correction budget exhausted | stops safely; nothing sent | TBD |
| Escalation (user insists on an opted-out number) | explained; stopped | TBD |
| Cancellation (user declines at S6) | `cancelled`, nothing sent | TBD |
| Rollback / compensation | Not applicable — a sent SMS cannot be recalled; a correction draft is offered (tested as a new run) | — |
| Output schema and status | valid `sms_result`, correct status | TBD |
| Compatibility / regression | passes on a new provider API version | TBD |

All tests use sandbox numbers owned by the tester or provider test credentials; never real recipients. Not production-ready until the mandatory tests have evidence — and the provider variant cannot be activated at all until it is built and its tests pass.

## Q. Open questions and known limitations

- Outbound SMS provider, opt-out register, inbound-reply handler and delivery-receipt handling are not built (owner: publisher).
- Skill Port / Vedaxi binding IDs, schema registry and approval UI — TBD (owner: publisher, with Skill Port).
- Per-message cost cap, daily caps and text retention — TBD (owner: operator).
- Country sender rules, opt-out keyword lists and deadlines, and quiet-hour laws change; they must be confirmed with the provider and counsel per destination before activation.
- The `sms:` link body parameter is handled differently by some phone platforms; hand-off may require the user to paste the text.
- Hand-off delivery cannot be confirmed by the agent.
