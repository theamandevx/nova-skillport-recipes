# Prepare a WhatsApp message

## A. Identity and ownership

- **Recipe ID:** `nova.comms.prepare-a-whatsapp-message` (proposed until assigned by Skill Port)
- **Name:** Prepare a WhatsApp message
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** productivity
- **Tags:** whatsapp, messaging, drafting, linked-device, business-api, templates, human-approval · Vedaxi task type: Prepare a WhatsApp message
- **Summary:** On an explicit request, the agent resolves the recipient, drafts one WhatsApp message, checks the channel (linked-device session health, recipient allowlist, or for business messaging the official API's opt-in, 24-hour window and template rules), and asks the user to approve the exact text. It sends only after a yes — through the user's paired linked-device session for personal messages, the WhatsApp Business Platform for customer-facing messages, or by handing the text to the user.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** An LLM for drafting; a contact store; an owner notification channel. Personal sends: a paired linked-device session through a multi-device bridge. Customer-facing sends: the WhatsApp Business Platform (Cloud API) with a registered business number, approved templates and an opt-in record. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — Nova's reference implementation has a working WhatsApp channel through a paired linked-device session and drafts and delivers messages on it after the user's yes. That session can drop (the linked device is removed or logged out) and then needs the owner to re-pair it. Sending to arbitrary third-party numbers through the session has not been verified there, and the Business Platform variant is not built. Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.contacts.directory` | Capability | Find a person's WhatsApp number from approved sources | medium | UNVERIFIED |
| `nova.contacts.directory.lookup` | Operation (read) | Look a name up in assistant memory, then the contact directory | medium | UNVERIFIED |
| `nova.comms.whatsapp.channel` | Capability | Channel readiness and messaging rules per variant | high | UNVERIFIED |
| `nova.comms.whatsapp.channel.status` | Operation (read) | Linked-device session state; Business Platform number readiness | low | UNVERIFIED |
| `nova.comms.whatsapp.channel.window` | Operation (read; business variant only) | Opt-in record and time since the customer's last inbound message | medium | UNVERIFIED — not built |
| `nova.comms.whatsapp.compose` | Capability | Draft and check one message | low | UNVERIFIED |
| `nova.comms.whatsapp.compose.draft` | Operation (read, LLM) | Free-form text from the user's intent | low | UNVERIFIED |
| `nova.comms.whatsapp.compose.fill_template` | Operation (read; business variant only) | Pick an approved template and fill its parameters | medium | UNVERIFIED — not built |
| `nova.comms.whatsapp.compose.check` | Operation (read; rules) | Length, forbidden content and added-fact checks | low | UNVERIFIED |
| `nova.comms.whatsapp.send` | Capability | Deliver one approved message | high | UNVERIFIED |
| `nova.comms.whatsapp.send.message` | Operation (external side effect, or human hand-off) | Send through the selected variant | high | UNVERIFIED |
| `nova.comms.whatsapp.status` | Capability | Delivery state from receipts | low | UNVERIFIED |
| `nova.comms.whatsapp.status.get` | Operation (read) | Return `accepted`, `sent`, `delivered`, `read` or `failed` | low | UNVERIFIED |
| `nova.notify.owner` | Capability | Deliver a message to the requesting user's own channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Show the draft, collect approval, report the result | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.contacts.directory.lookup.memory` → `.directory` → `.ask-user` — read-only memory first, read-only directory second, then ask the user once. Never guess a number.
- `nova.comms.whatsapp.send.message.linked-device` — the user's own WhatsApp account through a paired linked-device session (a multi-device bridge). For **personal, one-to-one, low-volume** messages only. Binding `binding://nova/comms/whatsapp-linked`, protocol: internal bridge; mode: async. Eligible only when the session reports `connected`, the recipient is on the user's outbound allowlist, and the daily cap is not reached. Automated or bulk messaging from a personal account can breach WhatsApp's terms and get the number banned, so this variant is never used for customer-facing or repeated sends.
- `nova.comms.whatsapp.send.message.business-api` — the WhatsApp Business Platform (Cloud API) with a registered business number. **Required for customer-facing messaging.** Free-form messages only inside the 24-hour customer service window (counted from the customer's last inbound message); outside it, only pre-approved template messages; business-initiated messages need the recipient's opt-in. Binding `binding://nova/comms/whatsapp-business` (UNVERIFIED, **not built**). Confirm current rules and pricing with the platform's documentation before activation.
- `nova.comms.whatsapp.send.message.handoff` — human-assisted: the agent returns the approved text and the platform's click-to-chat link with the number and text filled in; the user sends from their own phone. Binding `binding://nova/comms/whatsapp-handoff`. The agent cannot confirm delivery.
- `nova.comms.whatsapp.channel.status.default`, `nova.comms.whatsapp.channel.window.default`, `nova.comms.whatsapp.compose.draft.default`, `nova.comms.whatsapp.compose.fill_template.default`, `nova.comms.whatsapp.compose.check.default`, `nova.comms.whatsapp.status.get.default`, `nova.notify.owner.send.default` — only the default variant is supported.

**Variant selection policy:** filter by `purpose` first — `business` → `business-api` only (or `handoff` if the user chooses to send personally); `personal` → `linked-device`, else `handoff`. Then installed, authorized and healthy. A switch between variants changes the sender identity, rules or cost, so it is **never silent**: if the approved variant becomes unavailable at send time (session dropped, window closed), the agent stops and asks. The personal `linked-device` variant is never a fallback for `business-api`.

**Bindings:** `binding://nova/contacts/lookup`, `binding://nova/comms/whatsapp-status`, `binding://nova/comms/whatsapp-window`, `binding://nova/comms/whatsapp-compose`, `binding://nova/comms/whatsapp-template`, `binding://nova/comms/whatsapp-check`, `binding://nova/comms/whatsapp-linked`, `binding://nova/comms/whatsapp-business`, `binding://nova/comms/whatsapp-handoff`, `binding://nova/comms/whatsapp-receipts`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: the assistant runtime's WhatsApp channel runs a multi-device bridge paired to a WhatsApp account as a linked device; pairing is done by the owner scanning a QR code on their phone, and the pairing code is delivered to the owner only.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Contact | `schema://nova/prepare-a-whatsapp-message/contact@0.1.0` | the requesting user | read |
| Channel state | `schema://nova/prepare-a-whatsapp-message/channel-state@0.1.0` | the user's installation | read |
| Message draft | `schema://nova/prepare-a-whatsapp-message/wa-draft@0.1.0` | this run | create, update (before approval) |
| Outbound message | `schema://nova/prepare-a-whatsapp-message/wa-record@0.1.0` | the requesting user's tenant | create, read |

**Authorization requirements (declared, never a grant):** `contacts:read`, `whatsapp:channel:read`, `whatsapp:send:personal` (linked-device), `whatsapp:send:business` (Business Platform), `whatsapp:status:read`, `notify:owner`. Linked-device session credentials live in the runtime as `secret_ref://whatsapp-linked-session` (owner-only file permissions); Business Platform token as `secret_ref://whatsapp-business`; LLM key `secret_ref://llm`.

**Policy / trust:** risk class `high` — a message reaches a third party under the user's or the business's identity. Requires per-message approval, purpose-based variant rules, a recipient allowlist for personal sends, quiet hours, and for business sends the opt-in, 24-hour window and template rules. A pairing QR code is equivalent to account access and is shown only to the owner through an authenticated channel.

## C. Intent and applicability

- **User problem:** The user wants a WhatsApp message written and sent (or ready to send) without typing it, and wants control over what goes out under their name.
- **Desired outcome:** One approved message delivered through the right channel, with a truthful delivery state — or handed to the user ready to send.
- **Trigger or entry conditions:** The user explicitly asks to WhatsApp or message a named person or number and says what to say.
- **Applicable situations:** Personal one-to-one messages (confirmations, replies, short updates); a single customer-facing message through the Business Platform when installed (for example a reply to a customer who wrote in the last 24 hours, or an approved utility template).
- **Out-of-scope situations:** Group chats; broadcasts, bulk or marketing campaigns; media attachments, voice notes and calls (not in this version); messages to people who have not opted in to business messages; one-time codes; impersonation; re-pairing the linked device (owner-only, manual).
- **Required inputs:** `recipient`, `intent` (see Section D).
- **Prerequisites:** A contact source; an LLM; at least one send variant installed (hand-off is always available).
- **Required capabilities:** `nova.contacts.directory`, `nova.comms.whatsapp.channel`, `nova.comms.whatsapp.compose`, `nova.comms.whatsapp.send`, `nova.comms.whatsapp.status`, `nova.notify.owner` (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** The Business Platform variant (send, window check, templates, status webhooks) is not built in the reference implementation. Third-party sends through the linked-device session are unverified there.
- **Assumptions:** Verified in the reference implementation: a paired linked-device channel that delivers messages after the user's yes; the session can drop and needs owner re-pairing. Assumed: the runtime enforces a per-user outbound allowlist and reports session state.
- **Ambiguity policy:** Ask. If the recipient, content or purpose (personal or business) is unclear, ask one question; never guess.
- **Vedaxi task type:** Prepare a WhatsApp message.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `recipient` | string: a name, or an E.164 number (`^\+[1-9][0-9]{7,14}$`) | yes | user; personal data | — | Ask once. Invalid number: reject and ask. Two matches: ask which. |
| `intent` | string, 5–1,000 chars | yes | user; may be personal | — | Ask for it; never send without content from the user. |
| `exact_text` | string, ≤ 4,096 chars | no | user | — | Used verbatim (checked, not rewritten). |
| `purpose` | enum `personal`, `business` | no | user or inferred | `personal` | A customer or client recipient with `personal`: ask; never send customer-facing messages from the linked device. |
| `delivery_mode` | enum `auto`, `handoff` | no | user | `auto` | — |
| `template_name` | string | no (business, outside the window) | user or operator | — | Unknown or not approved: list approved templates and ask. |
| `language` | BCP 47 tag | no | user or inferred | the user's language | Business template not available in that language: ask. |

```json
{"$id":"schema://nova/prepare-a-whatsapp-message/request@0.1.0","type":"object","required":["recipient","intent"],
 "properties":{"recipient":{"type":"string","minLength":2},"intent":{"type":"string","minLength":5,"maxLength":1000},
 "exact_text":{"type":"string","maxLength":4096},"purpose":{"enum":["personal","business"]},
 "delivery_mode":{"enum":["auto","handoff"]},"template_name":{"type":"string"},"language":{"type":"string"}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `whatsapp_result` | object `schema://nova/prepare-a-whatsapp-message/whatsapp-result@0.1.0` | `status`, `purpose`, `delivery_mode`, `delivery_state`, `to_masked` | the requesting user's channel and the run record | personal data; owner only | approval hash, channel checks and receipts (Section I) |

```json
{"$id":"schema://nova/prepare-a-whatsapp-message/whatsapp-result@0.1.0","type":"object",
 "required":["status","purpose","delivery_mode","delivery_state","to_masked"],
 "properties":{"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},
 "purpose":{"enum":["personal","business"]},"text":{"type":["string","null"],"maxLength":4096},
 "template":{"type":["object","null"],"properties":{"name":{"type":"string"},"language":{"type":"string"},
   "params":{"type":"array","items":{"type":"string"}}}},
 "delivery_mode":{"enum":["linked_device","business_api","handoff","none"]},
 "delivery_state":{"enum":["not_sent","handed_off_unconfirmed","accepted","sent","delivered","read","failed"]},
 "to_masked":{"type":"string"},"message_id":{"type":["string","null"]},"approval_id":{"type":["string","null"]},
 "checks":{"type":"array","items":{"type":"object","required":["name","result"],
   "properties":{"name":{"type":"string"},"result":{"enum":["pass","fail","not_applicable"]},"reason":{"type":"string"}}}}}}
```

Helper schemas (draft, version 0.1.0; each `$id` matches the manifest reference):

```json
{"$id":"schema://nova/prepare-a-whatsapp-message/contact-query@0.1.0","type":"object","required":["query"],"properties":{"query":{"type":"string","minLength":2},"max_results":{"type":"integer","minimum":1,"maximum":5}}}
{"$id":"schema://nova/prepare-a-whatsapp-message/contact@0.1.0","type":"object","required":["name","phone_e164","source"],"properties":{"name":{"type":"string"},"phone_e164":{"type":"string","pattern":"^\\+[1-9][0-9]{7,14}$"},"country":{"type":"string","pattern":"^[A-Z]{2}$"},"time_zone":{"type":"string"},"source":{"enum":["memory","directory","user"]}}}
{"$id":"schema://nova/prepare-a-whatsapp-message/channel-query@0.1.0","type":"object","required":["variant","purpose"],"properties":{"variant":{"enum":["linked-device","business-api","handoff"]},"purpose":{"enum":["personal","business"]},"phone_e164":{"type":["string","null"],"pattern":"^\\+[1-9][0-9]{7,14}$"}}}
{"$id":"schema://nova/prepare-a-whatsapp-message/channel-state@0.1.0","type":"object","required":["variant","state","checked_at"],"properties":{"variant":{"enum":["linked-device","business-api","handoff"]},"state":{"enum":["connected","connecting","logged_out","not_installed","unknown"]},"checked_at":{"type":"string","format":"date-time"},"recipient_allowed":{"type":["boolean","null"]},"daily_sent":{"type":["integer","null"],"minimum":0},"opt_in":{"type":["boolean","null"]},"last_inbound_at":{"type":["string","null"],"format":"date-time"},"window_open":{"type":["boolean","null"]}}}
{"$id":"schema://nova/prepare-a-whatsapp-message/wa-draft@0.1.0","type":"object","required":["chars","content_sha256"],"oneOf":[{"required":["text"],"properties":{"text":{"type":"string"}}},{"required":["template"],"properties":{"template":{"type":"object"}}}],"properties":{"text":{"type":["string","null"],"maxLength":4096},"template":{"type":["object","null"],"required":["name","language","params"],"properties":{"name":{"type":"string"},"language":{"type":"string"},"params":{"type":"array","items":{"type":"string"}}}},"chars":{"type":"integer","minimum":0},"content_sha256":{"type":"string"}}}
{"$id":"schema://nova/prepare-a-whatsapp-message/send-input@0.1.0","type":"object","required":["variant","purpose","to_e164","content_sha256","approval_id","idempotency_key"],"properties":{"variant":{"enum":["linked-device","business-api","handoff"]},"purpose":{"enum":["personal","business"]},"to_e164":{"type":"string","pattern":"^\\+[1-9][0-9]{7,14}$"},"text":{"type":["string","null"],"maxLength":4096},"template":{"type":["object","null"],"required":["name","language","params"],"properties":{"name":{"type":"string"},"language":{"type":"string"},"params":{"type":"array","items":{"type":"string"}}}},"content_sha256":{"type":"string"},"approval_id":{"type":"string"},"idempotency_key":{"type":"string"}}}
{"$id":"schema://nova/prepare-a-whatsapp-message/wa-record@0.1.0","type":"object","required":["message_id","variant","to_e164","content_sha256","approval_id","state","state_at"],"properties":{"message_id":{"type":"string"},"variant":{"enum":["linked-device","business-api","handoff"]},"to_e164":{"type":"string","pattern":"^\\+[1-9][0-9]{7,14}$"},"content_sha256":{"type":"string"},"approval_id":{"type":"string"},"state":{"enum":["handed_off_unconfirmed","accepted","sent","delivered","read","failed"]},"state_at":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/prepare-a-whatsapp-message/wa-ref@0.1.0","type":"object","anyOf":[{"required":["message_id"]},{"required":["idempotency_key"]}],"properties":{"message_id":{"type":"string"},"idempotency_key":{"type":"string"}}}
{"$id":"schema://nova/prepare-a-whatsapp-message/delivery@0.1.0","type":"object","required":["message_id","status"],"properties":{"message_id":{"type":"string"},"status":{"enum":["delivered","failed"]},"delivered_at":{"type":["string","null"],"format":"date-time"},"channel":{"type":"string"}}}
```

**Example (fictional)**

Input: `{"recipient":"Jane Doe","intent":"Ask if Saturday lunch at 1 pm still works and say I'll book the table.","purpose":"personal"}`

Output: `{"status":"completed","purpose":"personal","text":"Hi Jane! Does Saturday lunch at 1 pm still work for you? If so I'll book the table.","template":null,"delivery_mode":"linked_device","delivery_state":"delivered","to_masked":"…0142","message_id":"wa_0007","approval_id":"apr_0102","checks":[{"name":"session","result":"pass"},{"name":"allowlist","result":"pass"},{"name":"quiet_hours","result":"pass"},{"name":"content","result":"pass"}]}`

## E. Constraints and budgets

- **Hard constraints:** explicit user request; per-message approval of the exact text or template and parameters (hash-bound) before any send or hand-off; content only from the user's intent — no invented facts, promises or links; customer-facing (`business`) messages only through the Business Platform or the user's own hand-off, never the linked-device session; Business Platform rules: opt-in before business-initiated messages, free-form only inside the 24-hour window, otherwise an approved template; linked-device sends only to allowlisted recipients, one-to-one, within the daily cap; never automate re-pairing, and show a pairing QR code only to the owner through an authenticated channel; never include passwords, one-time codes or full account or card numbers; never impersonate; one message per approval; no group or broadcast sends.
- **Soft constraints:** keep messages under 1,000 characters (platform maximum 4,096); quiet hours 09:00–20:00 recipient time for personal sends (the user may override for one message in the approval); the user's tone and language.
- **Resource and tenant scope:** the requesting user's contacts, channel installation and outbound records only.
- **Data sources allowed:** assistant memory, the user's contact directory, the user's own message; for business sends, the opt-in record and the conversation's last inbound timestamp.
- **External transfers permitted:** the intent to the configured LLM; the number and text to WhatsApp through the selected variant solely to deliver this message.
- **Time limit:** drafting ≤ 60 s; approval wait ≤ 30 minutes; receipt wait ≤ 10 minutes.
- **Cost or token budget:** linked-device and hand-off have no per-message fee; Business Platform pricing depends on message category and country (`TBD`); per-message cap `TBD`.
- **Maximum steps:** 9 (S1–S9).
- **Maximum retries and correction cycles:** send: 0 automatic re-sends; reads: 2 retries; redrafting: 1 automatic plus up to 3 user-requested edits; correction cycles: 1 per step, 3 per run.
- **Rate limits and concurrency:** linked-device: at most 1 message per recipient per 10 minutes and 20 per day (TBD); Business Platform messaging limits and quality rules apply.
- **Retention and logging constraints:** outbound records keep text hash, variant and state; text retained per operator policy (`TBD`); session credentials never logged.
- **Prohibited actions:** re-sending without approval; editing approved text; switching variant silently; messaging non-allowlisted recipients from the linked device; sending outside the window without a template; sending to a recipient who asked not to be messaged.
- **Stop conditions:** approval declined or timed out; session `logged_out` with no accepted alternative; no opt-in or no approved template for a business send; recipient not allowlisted and the user does not add them; the user cancels.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Message draft (in-run) | request valid | Allowed | S5 checks pass |
| Update | Draft text or template parameters (before approval only) | user asked for an edit | Allowed | re-checked |
| Create | Outbound message record (one per approval) | S3–S6 passed | Approval Required (S6) | record with message ID |
| Read | Contact, channel state, opt-in and window, outbound state | own tenant | Allowed | — |
| Update | Recipient allowlist (add one recipient) | the user explicitly asks while approving a personal send | Approval Required | allowlist re-read shows the entry |
| Delete | any | — | Not Allowed | — |
| External side effect | WhatsApp message to the recipient | approved hash, variant rules, quiet hours | Approval Required (S6) | receipt `delivered` or `read` |
| External side effect | hand-off text and link to the requesting user | approved hash | Approval Required (S6) | the user receives the hand-off |
| External side effect | result message to the requesting user | run finished | Allowed | delivery receipt |

- **Idempotency:** send key = hash(`run_id`, `to_e164`, `content_sha256` (text or template), `approval_id`); a repeated send with the same key returns the existing record.
- **Expected side effects:** the recipient receives the message under the user's (linked-device) or business's (Business Platform) identity; Business Platform charges where applicable.
- **Compensation:** WhatsApp offers delete-for-everyone for a limited time on personal accounts, but the recipient may already have read it; the agent does not use it automatically — it can draft a correction or, on the user's explicit request, attempt deletion where the variant supports it.
- **Irreversible:** a delivered or read message; charges.
- **Ambiguous outcomes:** if a send times out, read status by idempotency key (S8) before anything else; never re-send on an unknown outcome.

## G. Autonomy and approval policy

- **Default autonomy level:** A1 — supervised execution.
- **Planning level:** A2 (resolve, choose variant, draft, check).
- **Execution level:** A1 — sending only after the S6 approval; A0 for hand-off.
- **Verification level:** A2.
- **Recovery level:** A1 — no re-send or variant switch without a new approval.
- **Approval required for:** the exact text or template and parameters, recipient and variant (every message); adding a recipient to the allowlist; overriding personal quiet hours.
- **Actions never permitted:** sending without approval; customer-facing messages from the linked device; bulk, broadcast or group sends; sending outside the 24-hour window without an approved template; re-pairing the device; forwarding a pairing QR code to anyone but the owner; impersonation; secrets in messages.
- **Conditions requiring clarification:** ambiguous recipient; unclear purpose; missing template outside the window.
- **Conditions requiring escalation:** session logged out; repeated send failures; a recipient who asked not to be messaged.
- **Safe state after stop, cancellation or escalation:** nothing sent; draft kept in the run record; the user told why and what they can do (for example "re-pair WhatsApp from your phone: Linked devices").

Approval covers one message with the shown hash, recipient and variant. It does not cover re-sends, edits or other recipients.

## H. Execution plan

**S1 — Validate the request**
1. Objective: confirm recipient, intent and purpose.
2. Capability: none (local validation).
3. Input mapping: user message → `request@0.1.0`.
4. Dependencies: none.
5. Preconditions: explicit request to message someone on WhatsApp.
6. Action / side effects: parse and validate; infer `purpose`; no side effects.
7. Autonomy: A2; no approval.
8. Timeout / retry: 5 s / none.
9. Idempotency: not applicable.
10. Postconditions: schema-valid request with a purpose.
11. Verification: JSON Schema validation passes.
12. Failure routes: missing or invalid field → ask once → else `blocked`; group or broadcast request → explain it is out of scope.
13. Escalation triggers: none.
14. Evidence: validated request.

**S2 — Resolve the recipient**
1. Objective: exactly one E.164 WhatsApp number with country and time zone.
2. Capability: `nova.contacts.directory.lookup` (memory → directory → ask-user).
3. Input mapping: `recipient` → `query`.
4. Dependencies: S1.
5. Preconditions: `recipient` is a name (skip lookup if already a number).
6. Action / side effects: read-only lookups; asking the user is a message to the user only.
7. Autonomy: A2 for reads; A0 for ask-user.
8. Timeout / retry: 10 s per variant / 2 retries.
9. Idempotency: read-only.
10. Postconditions: one `contact@0.1.0`.
11. Verification: E.164 pattern; exactly one match.
12. Failure routes: several matches → ask which; none → ask-user; declined → `blocked`.
13. Escalation triggers: none.
14. Evidence: which source supplied the number.

**S3 — Channel and policy checks**
1. Objective: choose an eligible variant and know its rules.
2. Capability: `nova.comms.whatsapp.channel.status`; `nova.comms.whatsapp.channel.window` (business).
3. Input mapping: `phone_e164`, `purpose`, `delivery_mode`, time zone.
4. Dependencies: S2.
5. Preconditions: number resolved.
6. Action / side effects: personal — session state, allowlist membership, daily cap, quiet hours; business — Business Platform installed, opt-in present, window open or closed, templates available; no side effects.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 1 retry.
9. Idempotency: read-only.
10. Postconditions: `channel-state@0.1.0` and the selected variant, or a named block.
11. Verification: every hard check has a recorded result; `connected` is required for linked-device.
12. Failure routes: session `logged_out` → tell the user re-pairing is needed (owner scans a QR code on their phone) and offer hand-off; not allowlisted → offer to add the recipient (approval at S6) or hand-off; business without the Business Platform → offer hand-off or `blocked`; no opt-in → `blocked`.
13. Escalation triggers: the recipient previously asked not to be messaged → stop.
14. Evidence: check results and selected variant.

**S4 — Compose**
1. Objective: the message content.
2. Capability: `nova.comms.whatsapp.compose.draft` (free-form) or `.fill_template` (business, window closed); skipped when `exact_text` is given.
3. Input mapping: `intent`, recipient first name, user's display name, `language`, `template_name`.
4. Dependencies: S3.
5. Preconditions: a variant is selected.
6. Action / side effects: write the text, or select an approved template and fill parameters from the intent only; no side effects.
7. Autonomy: A2.
8. Timeout / retry: 30 s / 1 retry.
9. Idempotency: not applicable.
10. Postconditions: `wa-draft@0.1.0`.
11. Verification: see S5.
12. Failure routes: no approved template fits → list templates and ask → none suitable → `blocked`.
13. Escalation triggers: none.
14. Evidence: draft text or template and parameters, with hash.

**S5 — Check the draft**
1. Objective: the draft is within limits and says only what the user wants.
2. Capability: `nova.comms.whatsapp.compose.check`.
3. Input mapping: draft.
4. Dependencies: S4.
5. Preconditions: draft exists.
6. Action / side effects: length ≤ 4,096 (soft 1,000); no secrets, one-time codes or full account numbers; no facts, links or promises beyond the intent; template parameters within their allowed lengths; no side effects.
7. Autonomy: A2.
8. Timeout / retry: 5 s / none.
9. Idempotency: not applicable.
10. Postconditions: checked draft.
11. Verification: all checks pass.
12. Failure routes: fail → fix once → still failing → show the user the issue and ask.
13. Escalation triggers: none.
14. Evidence: check results.

**S6 — Approval checkpoint**
1. Objective: explicit approval for this exact message, recipient and variant.
2. Capability: `nova.notify.owner.send` (user interaction).
3. Input mapping: recipient name, masked number, text or template preview, variant ("from your WhatsApp" / "from the business number" / "you send it"), allowlist addition if needed, quiet-hours note.
4. Dependencies: S5.
5. Preconditions: draft checked.
6. Action / side effects: ask "Send this to Jane Doe (…0142) from your WhatsApp?" with Approve / Hand off to me / Edit / Cancel.
7. Autonomy: A0 — human decision.
8. Timeout / retry: wait up to 30 minutes, then `cancelled`.
9. Idempotency: the approval ID binds one hash, recipient and variant.
10. Postconditions: approval record with ID, hash, variant, timestamp, and any allowlist addition or quiet-hours override.
11. Verification: the approval references the current draft hash.
12. Failure routes: Edit → S4/S5 (up to 3 rounds); Cancel or timeout → `cancelled`.
13. Escalation triggers: none.
14. Evidence: approval ID, hash, variant, timestamp.

**S7 — Send or hand off**
1. Objective: deliver the approved message exactly once.
2. Capability: `nova.comms.whatsapp.send.message` (approved variant).
3. Input mapping: `phone_e164`, approved text or template, approval ID, idempotency key.
4. Dependencies: S6.
5. Preconditions: approval valid; hash unchanged; re-check session state (linked-device) or window and opt-in (business) and quiet hours.
6. Action / side effects: the recipient receives the message (linked-device or business), or the user receives the hand-off text and link.
7. Autonomy: A1 (A0 for hand-off), covered by the S6 approval.
8. Timeout / retry: 20 s; **no automatic re-send**.
9. Idempotency: key = hash(`run_id`, `to_e164`, content hash, `approval_id`).
10. Postconditions: a record with a message ID in `accepted` or `sent`, or the hand-off delivered.
11. Verification: the record's `to` and content hash equal the approved values.
12. Failure routes: re-check fails (session dropped, window closed) → stop and ask (hand-off or template); timeout → S8 status read by key; send error → `failed` with the reason.
13. Escalation triggers: repeated send failures or session drops → tell the user.
14. Evidence: message ID or hand-off message ID, timestamps.

**S8 — Confirm delivery state**
1. Objective: a truthful final state.
2. Capability: `nova.comms.whatsapp.status.get`.
3. Input mapping: message ID or idempotency key.
4. Dependencies: S7.
5. Preconditions: an agent send happened (skipped for hand-off, which stays `handed_off_unconfirmed`).
6. Action / side effects: read receipts; `accepted` and `sent` are never reported as delivered.
7. Autonomy: A2.
8. Timeout / retry: up to 10 minutes / 2 read retries.
9. Idempotency: read-only.
10. Postconditions: `delivered`, `read` or `failed`, or `sent` with no receipt by the deadline.
11. Verification: state comes from receipts, not from the send response.
12. Failure routes: no receipt → `sent` (unconfirmed) → `partially_completed`.
13. Escalation triggers: `failed` → tell the user and offer SMS or a call.
14. Evidence: state transitions with timestamps.

**S9 — Report**
1. Objective: tell the user what happened.
2. Capability: `nova.notify.owner.send`.
3. Input mapping: `whatsapp_result`.
4. Dependencies: S8 (or S7 for hand-off).
5. Preconditions: result assembled.
6. Action / side effects: one message to the requesting user.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 2 retries.
9. Idempotency: message key = `run_id`.
10. Postconditions: delivered.
11. Verification: delivery receipt.
12. Failure routes: delivery fails → result in the run record → `partially_completed`.
13. Escalation triggers: none.
14. Evidence: message ID.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Input validity | request matches `request@0.1.0` | validator | valid | ask once, then `blocked` |
| Purpose rule | `business` never sent through linked-device | variant record | never | stop before sending |
| Session truth | linked-device send only when state `connected` at S3 and S7 | channel state | `connected` | stop; offer hand-off |
| Allowlist | linked-device recipient allowlisted (or approved addition) | allowlist read | member | ask or hand-off |
| Business rules | opt-in present; free-form only with window open; else approved template | window check at S3 and S7 | consistent | `blocked` or template |
| Content identity | sent content hash = approved hash | record vs approval | equal | stop before sending |
| Content safety | no secrets, codes, added facts | S5 | none | fix and re-approve |
| Delivery truth | `delivered` or `read` only from receipts; hand-off never delivered | S8 | consistent | correct the result |
| Output schema | `whatsapp_result` valid | validator | valid | fix fields; never fake values |

A send call returning a message ID proves only that the message was accepted, not that it reached the recipient.

## J. Self-correction and recovery

- **Failure classes:** validation, ambiguous recipient, unclear purpose, session dropped, not allowlisted, no opt-in, window closed, no template, content failure, timeout, send error, missing receipt, permission denial.
- **Permitted corrections:** re-ask once; fix the draft once and re-approve; offer hand-off or a template (needs the user's choice); reconcile an ambiguous send by reading status.
- **Forbidden corrections:** re-send without approval; switch variant silently; use the linked device for business messages; re-pair the device automatically; edit approved text; report hand-off or `sent` as delivered.
- **Maximum correction cycles:** 1 per step; 3 per run.
- **Reverification required after correction:** yes — S5 and S6 after any content change; S3 checks again at S7.
- **Stop conditions:** correction budget exhausted; hard constraint fails.
- **Recovery or compensation:** none automatic; draft a correction for a new approval, or attempt delete-for-everyone only on explicit request where supported.
- **What must be reported:** variant used, final state, any check that failed, session drops, allowlist changes.

## K. Independent challenge

- **Assumptions to challenge:** the number is on WhatsApp and belongs to the intended person; the recipient is a personal contact, not a customer; the message reads as intended without context.
- **Alternative approach:** SMS or a call when the recipient is not on WhatsApp or delivery failed; email for long or formal content.
- **Contradictory evidence:** a contact note marking the person as a customer; a previous request not to be messaged; a recent failed delivery to the number.
- **Could mislead:** reporting `sent` as delivered; a business message sent from a personal account; a draft that commits the user to something they did not say.
- **Separate verifier:** S5 rule checks run independently of drafting; delivery state comes only from receipts; the purpose rule is enforced at S3 and again at S7.
- **Findings requiring disclosure:** variant used, session problems, unconfirmed hand-off, missing receipts, allowlist additions, quiet-hours overrides.

## L. Human escalation

- **Triggers:** ambiguous recipient or purpose; session logged out; recipient not allowlisted; business send without opt-in or template; repeated failures; a do-not-message request on record.
- **Why the agent cannot decide:** the sender identity, consent and wording that speaks for the user are the user's decisions; re-pairing requires the owner's phone.
- **Information presented:** recipient (masked), content, variant and why, check results, delivery state, re-pairing instructions when relevant.
- **Decision requested:** approve / edit / cancel; hand off instead; add the recipient to the allowlist; choose a template.
- **Options and consequences:** approving sends a message that may be read immediately and cannot reliably be recalled; hand-off leaves sending to the user.
- **Safe behaviour while waiting:** nothing sent.
- **If no response:** after 30 minutes the run ends `cancelled`.
- **Resume:** a new approval is required; S3 checks re-run first.

## M. Completion conditions

Mandatory: `whatsapp_result` valid; the approval hash equals the sent or handed-off content; purpose and variant rules held; delivery state truthful; limitations disclosed; evidence references present.

- `completed` — agent send: receipt shows `delivered` or `read`; hand-off: approved text and link delivered to the user (state `handed_off_unconfirmed`, reported as such).
- `partially_completed` — `sent` with no delivery receipt by the deadline (for example the recipient's phone is offline), or delivered but the report to the user failed.
- `blocked` — invalid input; session logged out, recipient not allowlisted, no opt-in, or window closed without an approved template, and no alternative accepted.
- `failed` — the send errored or the receipt shows `failed`.
- `cancelled` — the user cancelled or approval timed out.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome; `whatsapp_result`; step statuses S1–S9; verification results and failed checks; evidence references (approval ID, content hash, message ID, receipt timestamps, channel state); approvals and scope (including allowlist additions and quiet-hours overrides); side effects (message sent or handed off, charges) and reconciliation status; corrections performed; unresolved challenges; limitations (Business Platform variant not built; third-party linked-device sends unverified); escalation state; learning candidate. Never include session credentials, pairing codes or full phone numbers in shared channels.

## O. Learning and revision

- **Outcome metrics:** approval rate, edit rounds, variant mix, delivered and read rates, session-drop frequency, time to re-pair, blocks by reason.
- **Evidence retained:** run record, approval record and delivery states, per the retention policy.
- **Data minimisation:** metrics stored without message text or numbers.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}`.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review; no silent changes to trusted behaviour; the purpose rule and allowlist cannot be relaxed without review.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (personal, linked-device, test account) | approved message delivered; receipt recorded | UNVERIFIED on Skill Port; delivery to the owner observed in the reference implementation |
| Happy path (hand-off) | approved text and link handed to the user | TBD |
| Happy path (business, sandbox number, window open) | free-form reply delivered | TBD — variant not built |
| Missing intent | asks once, then `blocked` | TBD |
| Invalid number | rejected before channel checks | TBD |
| Ambiguous purpose (recipient marked as a customer) | asks; never uses the linked device for business | TBD |
| Session logged out | linked-device `blocked`; re-pair instructions; hand-off offered | TBD |
| Recipient not allowlisted | addition requested at approval, or hand-off | TBD |
| Business window closed, no template | `blocked` or template selection | TBD |
| No opt-in (business) | `blocked` | TBD |
| Missing capability (Business Platform absent, business purpose) | hand-off offered or `blocked` | TBD |
| Permission denied (`whatsapp:send:personal` missing) | hand-off only | TBD |
| Unauthorized scope (another tenant's channel) | denied | TBD |
| Tenant isolation | cannot read another tenant's records or session state | TBD |
| Timeout on send | status read by key; no second send | TBD |
| Retry on transient status read | bounded retries | TBD |
| Duplicate / replay | same key returns the existing record | TBD |
| False success (message ID returned, receipt `failed`) | `failed`, not delivered | TBD |
| Failed verification (draft adds a promise) | fixed and re-approved | TBD |
| Correction budget exhausted | stops safely; nothing sent | TBD |
| Escalation (session drops between approval and send) | stops and asks; no silent switch | TBD |
| Cancellation (user declines at S6) | `cancelled`, nothing sent | TBD |
| Rollback / compensation (user asks to retract) | correction drafted, or delete attempted only on explicit request where supported | TBD |
| Output schema and status | valid `whatsapp_result`, correct status | TBD |
| Compatibility / regression | passes on a new bridge or Business Platform API version | TBD |

All tests use test accounts and sandbox numbers owned by the tester; never real customers. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, schema registry and approval UI — TBD (owner: publisher, with Skill Port).
- Business Platform variant (send, window check, templates, status webhooks) not built; pricing and messaging limits per country `TBD` (owner: publisher, operator).
- Third-party sends through the linked-device session are unverified in the reference implementation; personal-account automation carries a risk of the number being restricted (owner: publisher).
- The linked-device session can drop at any time; re-pairing needs the owner's phone, so personal sends can be unavailable for hours.
- Daily caps, quiet-hours defaults and text retention — TBD (owner: operator).
- Media, voice notes and group messages are out of scope in this version.
