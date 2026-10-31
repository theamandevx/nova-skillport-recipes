# Daily "needs you" digest

## A. Identity and ownership

- **Recipe ID:** `nova.inbox.daily-needs-you-digest` (proposed until assigned by Skill Port)
- **Name:** Daily "needs you" digest
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** productivity
- **Tags:** email, digest, needs-reply, important-mail, morning-brief, owner-notification, send-then-mark
- **Summary:** Every morning the agent collects the triaged emails that are important or need a reply and that the owner has not been told about or dismissed, sends one plain-text digest to the owner's chat, and only then marks those emails as notified, so nothing is dropped or sent twice. If nothing needs the owner, no message is sent.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** Triage results from recipe 08 (`importance`, `needs_reply`, `summary`); the dismiss flag from recipe 11; the owner notification channel (reference: a Telegram bot to the owner's private chat); a scheduler (reference: n8n, daily cron in the owner's time zone); a digest record store (reference: PostgreSQL behind a security-definer function). All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — reference implementation status: Built. The daily digest ran in Nova's reference implementation with send-then-mark and the dismiss exclusion. The digest record and adapter-level idempotency described here are a hardening of that design and have not been observed. Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.mail.attention` | Capability | The set of triaged emails that need the owner | medium | UNVERIFIED |
| `nova.mail.attention.unnotified` | Operation (read) | Important or needs-reply emails in the look-back window, not dismissed, not yet notified; one per Message-ID | low | UNVERIFIED |
| `nova.mail.attention.mark_notified` | Operation (update) | Set the notified marker on the listed emails and all their copies | low | UNVERIFIED |
| `nova.mail.digest` | Capability | One digest record per owner per local date | low | UNVERIFIED |
| `nova.mail.digest.get` | Operation (read) | Read today's digest record by key | low | UNVERIFIED |
| `nova.mail.digest.prepare` | Operation (create) | Store the composed digest with its item IDs and content hash before sending | low | UNVERIFIED |
| `nova.mail.digest.complete` | Operation (update) | Record `sent`, `quiet` or `delivery_unknown`, with the channel message ID | low | UNVERIFIED |
| `nova.notify.owner` | Capability | Deliver a message to the owner's own channel | medium | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Send the digest with an idempotency key | medium | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.notify.owner.send.default` — plain text, no markup parsing (an unbalanced formatting character must never break delivery), with the adapter recording idempotency key → channel message ID. Only the default variant is supported. There is no fallback channel: switching to email or SMS would change who can see the content, so it needs the owner's configuration, not a runtime fallback.
- `nova.mail.attention.unnotified.default`, `nova.mail.attention.mark_notified.default`, `nova.mail.digest.get.default`, `nova.mail.digest.prepare.default`, `nova.mail.digest.complete.default` — only the default variant is supported.

**Bindings:** `binding://nova/mail/attention-unnotified`, `binding://nova/mail/attention-mark-notified`, `binding://nova/mail/digest-get`, `binding://nova/mail/digest-prepare`, `binding://nova/mail/digest-complete`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: a scheduled workflow reads a security-definer function, posts to the owner's chat bot and calls the mark function after a successful post.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Attention item (view over triaged message copies) | `schema://nova/daily-needs-you-digest/attention-item@0.1.0` | the owner's tenant | read; update (`attention_notified_at` only) |
| Digest record | `schema://nova/daily-needs-you-digest/digest-record@0.1.0` | owner + local date | create, read, update (status fields) |
| Owner message | `schema://nova/daily-needs-you-digest/digest-message@0.1.0` | owner channel | create |

**Authorization requirements (declared, never a grant):** `mail:attention:read`, `mail:attention:mark`, `mail:digest:write`, `notify:owner`. Channel credentials stay in the runtime as `secret_ref://owner-channel`.

**Policy / trust:** risk class `medium` — the digest carries senders, subjects and summaries of private mail to the owner's chat provider. Nothing reaches a third party.

## C. Intent and applicability

- **User problem:** The owner does not want to scan the inbox to learn what needs action today.
- **Desired outcome:** One short morning message listing every important or needs-reply email the owner has not yet been told about, each exactly once; silence when there is nothing.
- **Trigger or entry conditions:** Daily schedule (default 08:00 in the owner's time zone); manual "send my digest now".
- **Applicable situations:** Any mailbox triaged by recipe 08.
- **Out-of-scope situations:** Triage itself (recipe 08); drafting replies (recipe 10); real-time alerts per email; digests to anyone other than the owner; money and obligations alerts (owned by their own recipes).
- **Required inputs:** Configuration only (Section D).
- **Prerequisites:** Triage running; owner channel connected and verified; owner time zone set.
- **Required capabilities:** `nova.mail.attention`, `nova.mail.digest`, `nova.notify.owner` (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port; every binding is a dependency to resolve before activation.
- **Assumptions:** Verified in the reference implementation: grouping into Important and Needs reply, send-then-mark, dismissed items excluded, quiet when empty. Assumed: the channel adapter can record an idempotency key with the message ID.
- **Ambiguity policy:** Use a documented safe default: when unsure whether an item was delivered, leave it unmarked (a possible duplicate is better than a silent drop) and say so in the next digest.
- **Vedaxi task type:** none — not one of the seven task types supported today.

## D. Inputs and outputs

**Inputs (configuration)**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `send_time_local` | `HH:MM` | no | owner | `08:00` | invalid or inside quiet hours: reject configuration |
| `time_zone` | IANA zone name | yes | owner | — | missing: `blocked` (cannot define "today") |
| `lookback_hours` | integer 1–168 | no | owner | 72 | out of range: clamp and report |
| `max_items` | integer 1–30 | no | owner | 15 | out of range: clamp |
| `detail_level` | enum `full`, `minimal` | no | owner | `full` | unknown: `minimal` (shows less) |
| `dashboard_url` | URL | no | operator | none | invalid: omit links |

```json
{"$id":"schema://nova/daily-needs-you-digest/digest-config@0.1.0","type":"object","required":["time_zone"],"properties":{"send_time_local":{"type":"string","pattern":"^([01][0-9]|2[0-3]):[0-5][0-9]$"},"time_zone":{"type":"string","minLength":3},"lookback_hours":{"type":"integer","minimum":1,"maximum":168},"max_items":{"type":"integer","minimum":1,"maximum":30},"detail_level":{"enum":["full","minimal"]},"dashboard_url":{"type":"string","format":"uri"}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| digest message | `digest-message@0.1.0` | `digest_key`, `text`, `item_ids` | owner's chat | personal data; owner only | delivery receipt; listed IDs = marked IDs |
| `digest_result` | `digest-result@0.1.0` | `digest_key`, `status`, `listed`, `overflow`, `marked` | run record | counts only | counts reconcile |

```json
{"$id":"schema://nova/daily-needs-you-digest/digest-message@0.1.0","type":"object","required":["digest_key","text","item_ids"],"properties":{"digest_key":{"type":"string"},"text":{"type":"string","maxLength":3500},"item_ids":{"type":"array","items":{"type":"string"},"maxItems":30},"overflow":{"type":"integer","minimum":0}}}
```

```json
{"$id":"schema://nova/daily-needs-you-digest/digest-result@0.1.0","type":"object","required":["digest_key","status","listed","overflow","marked"],"properties":{"digest_key":{"type":"string"},"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},"outcome":{"enum":["sent","quiet","reconciled","delivery_unknown","not_sent"]},"listed":{"type":"integer","minimum":0},"overflow":{"type":"integer","minimum":0},"marked":{"type":"integer","minimum":0},"aged_out":{"type":"integer","minimum":0},"channel_message_id":{"type":["string","null"]}}}
```

Other schemas:

```json
{"$id":"schema://nova/daily-needs-you-digest/attention-query@0.1.0","type":"object","required":["lookback_hours"],"properties":{"lookback_hours":{"type":"integer","minimum":1,"maximum":168},"limit":{"type":"integer","minimum":1,"maximum":100}}}
{"$id":"schema://nova/daily-needs-you-digest/attention-item@0.1.0","type":"object","required":["item_id","rfc_message_id","from","subject","importance","needs_reply","summary","received_at"],"properties":{"item_id":{"type":"string"},"rfc_message_id":{"type":"string"},"mailbox_label":{"type":"string"},"from":{"type":"string"},"subject":{"type":"string"},"importance":{"enum":["high","normal","low"]},"needs_reply":{"type":"boolean"},"summary":{"type":"string","maxLength":160},"received_at":{"type":"string","format":"date-time"},"link":{"type":["string","null"]}}}
{"$id":"schema://nova/daily-needs-you-digest/attention-batch@0.1.0","type":"object","required":["items","aged_out","last_triage_at"],"properties":{"items":{"type":"array","items":{"$ref":"schema://nova/daily-needs-you-digest/attention-item@0.1.0"}},"aged_out":{"type":"integer","minimum":0},"last_triage_at":{"type":["string","null"],"format":"date-time"}}}
{"$id":"schema://nova/daily-needs-you-digest/digest-key@0.1.0","type":"object","required":["digest_key"],"properties":{"digest_key":{"type":"string","pattern":"^[a-z0-9_-]+:[0-9]{4}-[0-9]{2}-[0-9]{2}$"}}}
{"$id":"schema://nova/daily-needs-you-digest/digest-record@0.1.0","type":"object","required":["digest_key","status","item_ids","content_hash"],"properties":{"digest_key":{"type":"string"},"status":{"enum":["prepared","sent","quiet","delivery_unknown"]},"item_ids":{"type":"array","items":{"type":"string"}},"content_hash":{"type":"string"},"channel_message_id":{"type":["string","null"]},"prepared_at":{"type":"string","format":"date-time"},"sent_at":{"type":["string","null"],"format":"date-time"}}}
{"$id":"schema://nova/daily-needs-you-digest/mark-request@0.1.0","type":"object","required":["item_ids","digest_key"],"properties":{"item_ids":{"type":"array","items":{"type":"string"},"minItems":1},"digest_key":{"type":"string"}}}
{"$id":"schema://nova/daily-needs-you-digest/mark-result@0.1.0","type":"object","required":["marked","already_marked"],"properties":{"marked":{"type":"integer","minimum":0},"already_marked":{"type":"integer","minimum":0}}}
{"$id":"schema://nova/daily-needs-you-digest/delivery@0.1.0","type":"object","required":["delivered"],"properties":{"delivered":{"type":["boolean","null"]},"channel_message_id":{"type":["string","null"]},"deduplicated":{"type":"boolean"}}}
```

**Example (fictional)**

Message text:

```
Good morning. 2 emails need you.

IMPORTANT
1. Jane Doe — Contract renewal (reply needed)
   Asks for a Thursday call about renewing the contract. 18h ago.

NEEDS A REPLY
2. John Roe — Invoice question
   Asks which purchase order number to use. 6h ago.

Open the inbox view: https://dashboard.example.com/mail
```

Result: `{"digest_key":"owner_01:2026-03-03","status":"completed","outcome":"sent","listed":2,"overflow":0,"marked":2,"aged_out":0,"channel_message_id":"msg_7781"}`

## E. Constraints and budgets

- **Hard constraints:** owner's own channel only; mark an item notified only after the digest that lists it is confirmed delivered; never mark an item that was not listed; one digest per owner per local date (manual "send now" uses a separate key suffix); dismissed items never appear; plain text only; no LLM rewriting of summaries (the digest repeats stored triage text, so it cannot invent facts); never include bodies.
- **Soft constraints:** Important first, then Needs reply; newest first within a group; keep the message under 3,500 characters.
- **Resource and tenant scope:** the owner's triaged mail and digest records only.
- **Data sources allowed:** attention items and digest records.
- **External transfers permitted:** sender names, subjects (when `detail_level = full`) and summaries to the owner's chat provider.
- **Time limit:** run ≤ 2 minutes.
- **Cost or token budget:** no LLM calls; channel provider limits only.
- **Maximum steps:** 8 (S1–S8).
- **Maximum retries and correction cycles:** send 2 retries with the same idempotency key; mark 3 retries; 1 reconciliation cycle per run.
- **Rate limits and concurrency:** one run per owner at a time; channel rate limits apply.
- **Retention and logging constraints:** digest records 30 days (`TBD`); logs carry item IDs and counts, not subjects.
- **Prohibited actions:** sending to any other recipient; marking before delivery; resending a confirmed digest; reformatting with markup that can fail to parse.
- **Stop conditions:** time zone missing; channel not connected; owner channel authorization revoked.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | digest record (one per key) | items exist | Allowed | record readable by key |
| Read | attention items, digest records | own tenant | Allowed | — |
| Update | `attention_notified_at` on listed items and their copies | delivery confirmed | Allowed | listed IDs = marked IDs |
| Update | digest record `status`, `channel_message_id`, `sent_at` | after send or quiet | Allowed | status matches outcome |
| Update | triage fields, dismiss flag | — | Not Allowed | — |
| Delete | any | — | Not Allowed | — |
| External side effect | message to the owner's chat | items exist | Allowed (owner's own channel, configured at installation) | delivery receipt |

- **Idempotency:** `digest_key = <owner_id>:<local date>`; the adapter maps the key to one channel message; `mark_notified` is conditional on an empty marker.
- **Expected side effects:** one message in the owner's chat; markers set.
- **Compensation:** a wrongly marked item can be un-marked by the owner (it then appears in the next digest); a sent message can be deleted by the owner in the chat.
- **Irreversible:** the owner has seen the message.
- **Ambiguous outcomes:** if `send` times out, ask the adapter by key before any resend; if the adapter cannot tell, record `delivery_unknown`, leave items unmarked and flag a possible duplicate in the next digest.

## G. Autonomy and approval policy

- **Default autonomy level:** A2 — bounded autonomy (owner-only message).
- **Planning level:** A2.
- **Execution level:** A2.
- **Verification level:** A2.
- **Recovery level:** A2 — reconcile by key, never mark without confirmation.
- **Approval required for:** enabling the digest; changing the channel; changing `detail_level` to `full` when the channel is shared with others.
- **Actions never permitted:** sending to anyone but the owner; marking unlisted items; sending twice for a confirmed key.
- **Conditions requiring clarification:** the configured channel is a group chat (others would see the digest).
- **Conditions requiring escalation:** delivery failing on two consecutive days.
- **Safe state after stop, cancellation or escalation:** nothing marked unless delivery was confirmed; the digest record shows the true state.

## H. Execution plan

**S1 — Preflight and reconciliation**
1. Objective: decide whether to compose, reconcile a previous attempt, or stop.
2. Capability: `nova.mail.digest.get`.
3. Input mapping: owner ID + local date in `time_zone` → `digest_key`.
4. Dependencies: none.
5. Preconditions: valid configuration; channel connected.
6. Action / side effects: read the digest record; no side effects.
7. Autonomy: A2; no approval.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: one of `none`, `prepared`, `sent`, `quiet`, `delivery_unknown`.
11. Verification: at most one record per key.
12. Failure routes: `sent` with items unmarked → go to S7 (mark only); `sent` and marked or `quiet` → `completed`, nothing to do; `prepared` or `delivery_unknown` → ask the adapter by key: delivered → S7, not delivered → S6 with the stored text, unknown → S6 with a "may repeat" note.
13. Escalation triggers: none.
14. Evidence: record state found.

**S2 — Collect unnotified attention items**
1. Objective: list every item the owner still needs to hear about.
2. Capability: `nova.mail.attention.unnotified`.
3. Input mapping: `lookback_hours`.
4. Dependencies: S1.
5. Preconditions: no confirmed digest for this key.
6. Action / side effects: read items with `importance = high` or `needs_reply = true`, received within the look-back window, not dismissed, notified marker empty; one per Message-ID.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: an `attention-batch@0.1.0`: the items, a count of items older than the window that were never notified (`aged_out`), and the time triage last ran (`last_triage_at`).
11. Verification: no dismissed item; no item with a notified marker; distinct Message-IDs.
12. Failure routes: store error → `failed`, nothing sent.
13. Escalation triggers: none.
14. Evidence: item IDs and `aged_out` count.

**S3 — Quiet check**
1. Objective: send nothing when nothing needs the owner.
2. Capability: `nova.mail.digest.complete` (status `quiet`).
3. Input mapping: item count.
4. Dependencies: S2.
5. Preconditions: item count is 0.
6. Action / side effects: if `last_triage_at` is within the last 24 hours, record `quiet` and send no message. If triage is stale or has never run, do not stay silent: compose a one-line note ("No digest today: mail triage has not run since …") and continue at S5 with no item IDs.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: keyed by `digest_key`.
10. Postconditions: record `quiet`, or a stale-triage note queued for S5–S6.
11. Verification: on a true quiet day no channel call was made.
12. Failure routes: record write fails → `completed` anyway (nothing was sent) with a logged warning.
13. Escalation triggers: none.
14. Evidence: quiet record.

**S4 — Compose the digest**
1. Objective: a short, readable, plain-text message.
2. Capability: local template (no LLM).
3. Input mapping: items, `max_items`, `detail_level`, `dashboard_url`.
4. Dependencies: S2.
5. Preconditions: at least one item.
6. Action / side effects: group Important (high, with a "reply needed" note when both) then Needs reply; newest first; each line = sender, subject (truncated to 80 characters; omitted when `minimal`), stored summary, age, link; overflow beyond `max_items` or 3,500 characters becomes "+K more — open the inbox view"; strip control characters.
7. Autonomy: A2.
8. Timeout / retry: 2 s / none.
9. Idempotency: deterministic for the same items.
10. Postconditions: `digest-message@0.1.0` with `item_ids` = listed items only.
11. Verification: text ≤ 3,500 characters; every listed ID appears once; summaries copied verbatim.
12. Failure routes: over length after truncation → reduce `max_items` and recompose once.
13. Escalation triggers: none.
14. Evidence: content hash and listed IDs.

**S5 — Prepare the digest record**
1. Objective: persist what will be sent before sending it.
2. Capability: `nova.mail.digest.prepare`.
3. Input mapping: `digest_key`, item IDs, content hash, text.
4. Dependencies: S4.
5. Preconditions: no confirmed record for the key.
6. Action / side effects: create a record with status `prepared`.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: unique on `digest_key`; an existing `prepared` record is reused only if its hash matches.
10. Postconditions: record `prepared`.
11. Verification: read back by key.
12. Failure routes: write fails → do not send; `failed`.
13. Escalation triggers: none.
14. Evidence: record ID.

**S6 — Send to the owner**
1. Objective: deliver the digest exactly once.
2. Capability: `nova.notify.owner.send.default`.
3. Input mapping: text, `digest_key` as idempotency key.
4. Dependencies: S5 (or S1 reconciliation).
5. Preconditions: record `prepared` or `delivery_unknown`.
6. Action / side effects: one message to the owner's chat.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 2 retries with the same key.
9. Idempotency: the adapter returns the existing message ID for a repeated key (`deduplicated: true`).
10. Postconditions: a channel message ID, or `delivery_unknown`.
11. Verification: delivery receipt with a message ID.
12. Failure routes: definite failure → record stays `prepared`, items unmarked, `failed` (next run retries); unknown → `delivery_unknown`, items unmarked, `partially_completed`.
13. Escalation triggers: failing on two consecutive days → escalate (Section L).
14. Evidence: channel message ID or error class.

**S7 — Mark listed items notified**
1. Objective: make sure listed items are not repeated tomorrow.
2. Capability: `nova.mail.attention.mark_notified`; `nova.mail.digest.complete` (status `sent`).
3. Input mapping: listed `item_ids`, `digest_key`.
4. Dependencies: S6 delivered (or S1 found `sent`).
5. Preconditions: delivery confirmed.
6. Action / side effects: set `attention_notified_at` on the listed items and every copy with the same Message-ID; record `sent` with the message ID.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 3 retries.
9. Idempotency: conditional on an empty marker; repeat calls report `already_marked`.
10. Postconditions: `marked + already_marked` equals listed items.
11. Verification: re-read listed items; all carry the marker; overflow items do not.
12. Failure routes: retries exhausted → record stays `sent` with items unmarked; the next run's S1 marks without resending.
13. Escalation triggers: none.
14. Evidence: `mark-result@0.1.0`.

**S8 — Verify and record the result**
1. Objective: an honest result for the run.
2. Capability: local assembly; run log.
3. Input mapping: outcomes of S1–S7.
4. Dependencies: S7, S3 or a failure route.
5. Preconditions: none.
6. Action / side effects: write `digest-result@0.1.0`.
7. Autonomy: A2.
8. Timeout / retry: 5 s / 1 retry.
9. Idempotency: keyed by run ID.
10. Postconditions: result stored.
11. Verification: listed = marked for `sent`; listed = 0 for `quiet`; `aged_out` disclosed.
12. Failure routes: mismatch → `partially_completed` with the gap named.
13. Escalation triggers: none.
14. Evidence: result record.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Configuration | config matches `digest-config@0.1.0`; send time outside quiet hours | validator | valid | `blocked` |
| Recipient | destination is the owner's verified private channel | channel binding | owner only | `blocked`, escalate |
| Exclusion | no dismissed or already-notified item is listed | S2 output vs flags | none | drop the item, report |
| Verbatim summaries | each listed summary equals the stored triage summary | S4 output vs store | equal | recompose |
| Send-then-mark order | markers set only after a delivery receipt | timestamps | marked_at > sent_at | un-mark, report |
| Listed = marked | marked IDs equal listed IDs | S7 result | equal | retry mark; else report |
| Single delivery | one channel message per `digest_key` | adapter key map | one | report the duplicate |
| Quiet truth | no channel call when items = 0 and triage is fresh; a stale-triage note when triage is stale | run trace, `last_triage_at` | as stated | report |

A channel returning success proves the message was accepted, not that the owner read it; the recipe claims delivery only.

## J. Self-correction and recovery

- **Failure classes:** invalid configuration, channel not connected, channel transient error, ambiguous delivery, store error, mark failure, length overflow.
- **Permitted corrections:** reconcile by `digest_key`; resend stored text with the same key; retry marking; shrink `max_items` to fit.
- **Forbidden corrections:** marking without delivery; sending to another channel; rewriting summaries; dropping items to make the run pass.
- **Maximum correction cycles:** 1 reconciliation and 1 recomposition per run.
- **Reverification required after correction:** yes — S7 re-reads markers; S4 checks length again.
- **Stop conditions:** missing time zone; revoked channel authorization.
- **Recovery or compensation procedure:** the owner can un-mark items; the next run repeats unconfirmed items.
- **What must be reported to the user:** possible duplicates, aged-out items and any delivery failure.

## K. Independent challenge

- **Assumptions to challenge:** that every listed item still needs the owner (it may have been answered from another device after triage).
- **Alternative approach:** a dashboard badge alone might suffice for owners who dislike chat messages; offer a weekly mode.
- **Contradictory evidence:** items the owner dismisses within minutes of the digest suggest triage over-flags; feed the rate back to recipe 08.
- **Could mislead:** an empty digest when triage is broken; S3 checks `last_triage_at` and sends a stale-triage note instead of staying quiet.
- **Separate verifier:** a count check between attention items in the store and the listed plus overflow count.
- **Findings requiring disclosure:** triage stale, aged-out items, delivery uncertainty.

## L. Human escalation

- **Triggers:** delivery failing on two consecutive days; the channel turns out to be shared; triage stale for more than 24 hours.
- **Why the agent cannot decide:** reconnecting or changing the channel changes who sees private mail details.
- **Information presented:** last successful digest, failure class, pending item count (no subjects outside the owner's private view).
- **Decision requested:** reconnect the channel, switch to `minimal`, pause the digest.
- **Options and consequences:** pausing means the owner relies on the dashboard; `minimal` hides subjects.
- **Safe behaviour while waiting:** items stay unmarked; the dashboard keeps showing them.
- **If no response:** daily attempts continue; one reminder per week.
- **Resume:** S1 reconciles by key before anything is sent.

## M. Completion conditions

Mandatory: digest record matches the outcome; listed IDs equal marked IDs for a sent digest; no dismissed item listed; no message when there was nothing; limitations disclosed; evidence references present.

- `completed` — digest delivered and every listed item marked, or a correct quiet day with fresh triage, or a previous digest fully reconciled.
- `partially_completed` — delivery unknown, marking incomplete, overflow left items for tomorrow, or triage was stale (the stale-triage note was sent instead of a digest).
- `blocked` — configuration or channel connection prevented any attempt.
- `failed` — delivery definitely failed or the store was unavailable; nothing marked.
- `cancelled` — the owner paused the digest before sending.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome ("2 items sent and marked"); `digest-result@0.1.0`; step statuses S1–S8; verification results and failed checks; evidence references (digest record, channel message ID, mark result); approvals in force (digest enablement, channel); side effects (one owner message, markers set) and reconciliation status; corrections and retries; unresolved challenges; limitations (overflow, aged-out, possible duplicate); escalation state; learning candidate. Never include bodies or credentials.

## O. Learning and revision

- **Outcome metrics:** delivery success rate, quiet-day rate, average items per digest, overflow rate, aged-out count, dismiss-after-digest rate, duplicate incidents.
- **Evidence retained:** digest records and results, per the retention policy.
- **Data minimisation:** metrics use counts; digest text is not kept in metrics.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}`.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (3 fictional items) | one message, 3 items marked | UNVERIFIED on Skill Port; observed in the reference implementation |
| Quiet day | no message, record `quiet` | UNVERIFIED on Skill Port; observed in the reference implementation |
| Quiet day with stale triage | one-line stale-triage note, `partially_completed` | TBD |
| Invalid input (bad time zone) | `blocked` | TBD |
| Missing input (no time zone) | `blocked` | TBD |
| Missing capability (channel binding absent) | `blocked`; nothing marked | TBD |
| Permission denied (`notify:owner` missing) | `blocked` | TBD |
| Unauthorized scope (channel is a group chat) | clarification requested; not sent | TBD |
| Tenant isolation | another tenant's items never listed | TBD |
| Dismissed item | never listed | TBD |
| Timeout on send, adapter confirms delivery | marked, no resend | TBD |
| Timeout on send, adapter unknown | items unmarked, `delivery_unknown` | TBD |
| Duplicate / replay (run twice same day) | second run sends nothing | TBD |
| Mark fails after send | next run marks without resending | TBD |
| False success (channel accepts but returns no message ID) | treated as unknown, not marked | TBD |
| Failed verification (summary altered) | recomposed with stored text | TBD |
| Overflow (40 items) | 15 listed, "+25 more", only 15 marked | TBD |
| Correction budget exhausted | stops; status honest | TBD |
| Escalation (two failed days) | owner alerted by the fallback path configured at installation, or dashboard banner | TBD |
| Cancellation (owner pauses) | `cancelled`; nothing sent | TBD |
| Compensation (owner un-marks) | item reappears next day | TBD |
| Output schema and status | valid `digest-result` | TBD |
| Compatibility / regression (channel API change) | plain-text send still works | TBD |

All tests use a test channel owned by the tester and fictional mail. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs and whether the platform's notification adapter supports idempotency keys — TBD (owner: publisher, with Skill Port).
- Whether to drop items whose thread the owner has already answered since triage — open design decision.
- Escalation path when the only channel is the one failing — TBD (dashboard banner proposed).
- Retention of digest records — TBD (owner: operator).
- The chat provider stores message content under its own terms; `minimal` mode reduces what it sees.
