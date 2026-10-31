# LinkedIn post in your voice

## A. Identity and ownership

- **Recipe ID:** `nova.content.linkedin-post-in-your-voice` (proposed until assigned by Skill Port)
- **Name:** LinkedIn post in your voice
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** marketing
- **Tags:** linkedin, writing, personal-brand, voice, approval-queue, human-approval, publishing-blocked · Vedaxi task type: none of the seven supported today maps directly (closest: Save a note draft)
- **Summary:** From a topic the user picked (or a short brief), the agent drafts a LinkedIn post in the user's own voice, checks it, and saves it to an approval queue where the user can edit, approve, reject or schedule it. A post is published only after a per-post approval, and only through the user's own approved LinkedIn app connection. Feature status: **Built, blocked** — drafting works; publishing is blocked until the user's LinkedIn app and OAuth connection are approved and active.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** A voice profile (style guide plus the user's example posts); a pro-tier LLM (medium variant optional); a content approval queue; an HTTP fetcher for source articles; the topic store from `nova.knowledge.trending-topics-radar` (optional); for publishing, the user's own LinkedIn developer app with member-posting access and an active OAuth grant. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — drafting in voice, the approval queue and due-date reminders for scheduled posts run in Nova's reference implementation. Publishing is **blocked**: it depends on the user's LinkedIn app and OAuth grant; in the reference implementation a member access token without refresh capability expired after about 60 days, and publishing stays blocked until the user reconnects or the app is approved for longer-lived access. Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.content.voice` | Capability | Hold the user's writing voice | low | UNVERIFIED |
| `nova.content.voice.get_profile` | Operation (read) | Return the style guide, example posts, banned phrases and brand colours | low | UNVERIFIED |
| `nova.topics.store` | Capability | Topic candidates picked by the user | low | UNVERIFIED |
| `nova.topics.store.get` | Operation (read) | Return one topic (title, URL, snippet) | low | UNVERIFIED |
| `nova.research.fetch` | Capability | Read a public article | low | UNVERIFIED |
| `nova.research.fetch.get` | Operation (read) | Fetch a URL as plain text | low | UNVERIFIED |
| `nova.content.draft` | Capability | Draft platform posts | medium | UNVERIFIED |
| `nova.content.draft.write` | Operation (read, LLM) | Draft a LinkedIn post (hook + body) and an optional image prompt in the user's voice | medium | UNVERIFIED |
| `nova.content.verify` | Capability | Check drafts before they reach the queue | low | UNVERIFIED |
| `nova.content.verify.format` | Operation (read) | Platform rules: lengths, hashtags, links, banned phrases | low | UNVERIFIED |
| `nova.content.verify.claims` | Operation (read) | Every factual claim traces to the source or the user's own brief | low | UNVERIFIED |
| `nova.content.queue` | Capability | The user's content approval queue | medium | UNVERIFIED |
| `nova.content.queue.save_draft` | Operation (create) | Save a draft, idempotent on a client key | low | UNVERIFIED |
| `nova.content.queue.edit` | Operation (update) | Change hook, body or image prompt of a non-published post | low | UNVERIFIED |
| `nova.content.queue.set_status` | Operation (update) | Move a post between `draft`, `approved`, `scheduled`, `published`, `rejected` | medium | UNVERIFIED |
| `nova.content.publish` | Capability | Publish an approved post to the user's own profile | high | UNVERIFIED (blocked) |
| `nova.content.publish.check_connection` | Operation (read) | Report whether the LinkedIn grant is active and has the posting scope | low | UNVERIFIED |
| `nova.content.publish.linkedin` | Operation (external side effect) | Create one post on the user's LinkedIn profile | high | UNVERIFIED (blocked) |
| `nova.content.due` | Capability | Remind the user when a scheduled post is due | low | UNVERIFIED |
| `nova.content.due.notify` | Operation (external side effect, owner only) | Notify the user; never auto-publish | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.content.draft.write.pro` — default; a reasoning-capable top-tier model, as in the reference implementation; given output-token headroom (about 4,000) because reasoning models can otherwise return empty content; an empty completion is a failure.
- `nova.content.draft.write.medium` — explicit choice only; lower cost; the result records the tier.
- The cheap tier is **not declared** for public-facing writing. There is no automatic tier switch: if the selected tier is unavailable, drafting fails with the reason and the user may choose another tier.
- Draft output format: caption and image prompt separated by a fixed delimiter line rather than JSON inside text, because multi-line captions make JSON-inside-text fragile to parse.
- `nova.content.verify.format.linkedin` — the only format profile in this recipe (see Section E for the rules).
- `nova.content.verify.claims.rules` — default; deterministic. `nova.content.verify.claims.llm-check` — optional second pass on a model different from the writer; can only flag, never approve.
- `nova.content.publish.linkedin.member-post` — the user's own LinkedIn app via OAuth; posts as the member (personal profile). Requires LinkedIn's member-posting permission (named `w_member_social` in LinkedIn's API at the time of writing — confirm against LinkedIn's current documentation). Company-page posting is not declared. There is **no alternative publish path**: no browser automation, no other account, no stored password.
- All other operations: only the default variant is supported.

**Bindings:** `binding://nova/content/voice`, `binding://nova/topics/store-get`, `binding://nova/research/fetch`, `binding://nova/content/draft`, `binding://nova/content/verify-format`, `binding://nova/content/verify-claims`, `binding://nova/content/queue-save`, `binding://nova/content/queue-edit`, `binding://nova/content/queue-status`, `binding://nova/content/linkedin-connection`, `binding://nova/content/linkedin-publish`, `binding://nova/content/due-notify` — all `status: UNVERIFIED`. Reference implementation: a content schema in PostgreSQL behind a security-definer dispatcher (save, list, get, edit, set-status) with a client-key idempotency column; an n8n draft gateway (header-authenticated webhook → model call with the voice guide as system prompt → parse → save); an n8n publish workflow (member info → create post → return the post URL); a five-minute n8n job that notifies the owner on Telegram when a scheduled post is due (send, then mark).

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Voice profile | `schema://nova/linkedin-post-in-your-voice/voice-profile@0.1.0` | the user | read |
| Topic | `schema://nova/trending-topics-radar/topic@0.1.0` | the user's tenant | read |
| Post | `schema://nova/linkedin-post-in-your-voice/post@0.1.0` | the user's tenant | create, read, update (content before publish; status) |
| Approval record | `schema://nova/linkedin-post-in-your-voice/approval@0.1.0` | linked to one Post and content hash | create |
| LinkedIn post | external, referenced by URL/URN | the user's LinkedIn profile | create (approval required) |

**Authorization requirements (declared, never a grant):** `content:voice:read`, `topics:read`, `web:fetch:public`, `llm:invoke:pro` or `llm:invoke:medium`, `content:queue:write`, `content:queue:read`, `content:publish:linkedin` (per-post approval), `notify:owner`. Provider grant: the user's LinkedIn OAuth token held by the runtime as `secret_ref://linkedin-oauth`; app credentials as `secret_ref://linkedin-app`; never in the recipe.

**Policy / trust:** risk class `high` for publishing (public, under the user's name, not reversible by the agent); `medium` for drafting. Source articles are untrusted data.

## C. Intent and applicability

- **User problem:** The user wants a steady LinkedIn presence in their own voice but lacks time to write each post.
- **Desired outcome:** A ready-to-approve post that sounds like the user, states only true, sourced facts, and goes live only when the user says so.
- **Trigger or entry conditions:** The user picks a topic (for example from the daily brief) or sends a brief and asks for a LinkedIn post.
- **Applicable situations:** Commentary on industry news, lessons learned, opinions the user supplies.
- **Out-of-scope situations:** Announcing the user's own news (use `nova.content.announcement-post`); multi-platform sets (use `nova.content.one-source-many-posts`); comments, direct messages, connection requests or any outreach; company-page posting; engagement automation; ghost-writing for someone else's profile.
- **Required inputs:** `topic_id` or `brief` (see Section D).
- **Prerequisites:** a voice profile (works without examples, flagged "voice not tuned"); a ready pro or medium model; the content queue. For publishing only: the user's LinkedIn app with member-posting access and an active OAuth grant.
- **Required capabilities:** `nova.content.voice`, `nova.content.draft`, `nova.content.verify`, `nova.content.queue`; optional `nova.topics.store`, `nova.research.fetch`, `nova.content.publish`, `nova.content.due` (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** Publishing is blocked until the user's LinkedIn app and grant are approved and active. Image generation is not part of this recipe (only an image prompt is produced).
- **Assumptions:** Verified in the reference implementation: drafting, queue, scheduling reminders. Assumed: the user's LinkedIn app can obtain member-posting access in their region.
- **Ambiguity policy:** Ask once when the angle or the topic is unclear; never invent the user's opinion, experience or results.
- **Vedaxi task type:** None maps directly; the closest supported type is Save a note draft.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `intent` | enum `draft_only`, `draft_and_publish` | no | user | `draft_only` | — |
| `topic_id` | string | one of `topic_id` / `brief` | topic store | — | Unknown or another tenant's: reject. |
| `brief` | string, 20–2,000 chars | one of `topic_id` / `brief` | user; may be confidential | — | Too short: ask for more. |
| `source_url` | URI (`http`/`https`) | no | user or topic | the topic's URL | Unreadable: draft from the brief only and disclose. |
| `angle` | string ≤ 300 chars | no | user (the user's own take) | none | None: the post states facts and a neutral takeaway; no invented opinion. |
| `model_tier` | enum `medium`, `pro` | no | user / operator | `pro` | Not ready: fail with the reason; no switch. |
| `include_image_prompt` | boolean | no | user | `true` | — |
| `schedule_for` | date-time with offset | no | user | none | In the past: reject; scheduled posts get a reminder, not auto-publish. |

Reused schemas (acknowledged): `topic@0.1.0` is defined in Section D of `nova.knowledge.trending-topics-radar` (folder `30-trending-topics-radar`) and used here unchanged. Own schemas:

```json
{"$id":"schema://nova/linkedin-post-in-your-voice/request@0.1.0","type":"object",
 "properties":{"intent":{"enum":["draft_only","draft_and_publish"]},"topic_id":{"type":"string"},
 "brief":{"type":"string","minLength":20,"maxLength":2000},"source_url":{"type":"string","format":"uri","pattern":"^https?://"},
 "angle":{"type":"string","maxLength":300},"model_tier":{"enum":["medium","pro"]},
 "include_image_prompt":{"type":"boolean"},"schedule_for":{"type":"string","format":"date-time"}},
 "anyOf":[{"required":["topic_id"]},{"required":["brief"]}]}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `post_result` | `schema://nova/linkedin-post-in-your-voice/post-result@0.1.0` | `post_id`, `status`, `queue_status`, `hook`, `body`, `publish` | the user's conversation; the queue; run record | private until published | format and claim checks (S5); publish confirmation (S10) |

```json
{"$id":"schema://nova/linkedin-post-in-your-voice/post-result@0.1.0","type":"object",
 "required":["post_id","status","queue_status","hook","body","publish"],
 "properties":{"post_id":{"type":"string"},"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},
 "queue_status":{"enum":["draft","approved","scheduled","published","rejected"]},"platform":{"const":"linkedin"},
 "hook":{"type":"string","maxLength":210},"body":{"type":"string"},"char_count":{"type":"integer","minimum":0},
 "hashtags":{"type":"array","maxItems":3,"items":{"type":"string","pattern":"^#[A-Za-z0-9_]+$"}},
 "suggested_first_comment":{"type":["string","null"]},"image_prompt":{"type":["string","null"]},
 "sources":{"type":"array","items":{"type":"string","format":"uri"}},"voice_tuned":{"type":"boolean"},"model_tier":{"enum":["medium","pro"]},
 "publish":{"type":"object","required":["state"],"properties":{"state":{"enum":["not_requested","blocked","published","unknown","failed"]},
  "url":{"type":["string","null"],"format":"uri"},"reason":{"type":["string","null"]}}}}}
```

Supporting schemas (compact, one per line):

```json
{"$id":"schema://nova/linkedin-post-in-your-voice/voice-query@0.1.0","type":"object","required":["tenant_id"],"properties":{"tenant_id":{"type":"string"},"platform":{"enum":["linkedin","x"]}}}
{"$id":"schema://nova/linkedin-post-in-your-voice/voice-profile@0.1.0","type":"object","required":["guide"],"properties":{"guide":{"type":"string","minLength":1},"examples":{"type":"array","maxItems":5,"items":{"type":"string"}},"banned_phrases":{"type":"array","items":{"type":"string"}},"brand_colours":{"type":"array","items":{"type":"string","pattern":"^#[0-9A-Fa-f]{6}$"}},"version":{"type":"string"}}}
{"$id":"schema://nova/linkedin-post-in-your-voice/fetch-input@0.1.0","type":"object","required":["url"],"properties":{"url":{"type":"string","format":"uri","pattern":"^https?://"},"max_bytes":{"type":"integer","minimum":1,"maximum":2097152}}}
{"$id":"schema://nova/linkedin-post-in-your-voice/page@0.1.0","type":"object","required":["url","status","usable"],"properties":{"url":{"type":"string","format":"uri"},"final_url":{"type":"string","format":"uri"},"status":{"type":"integer"},"text":{"type":["string","null"]},"fetched_at":{"type":"string","format":"date-time"},"content_hash":{"type":"string"},"usable":{"type":"boolean"}}}
{"$id":"schema://nova/linkedin-post-in-your-voice/post@0.1.0","type":"object","required":["post_id","platform","hook","body","status","content_hash"],"properties":{"post_id":{"type":"string"},"tenant_id":{"type":"string"},"platform":{"const":"linkedin"},"hook":{"type":"string","maxLength":210},"body":{"type":"string"},"image_prompt":{"type":["string","null"]},"source_type":{"enum":["topic","brief"]},"source_ref":{"type":["string","null"]},"status":{"enum":["draft","approved","scheduled","published","rejected"]},"scheduled_for":{"type":["string","null"],"format":"date-time"},"published_url":{"type":["string","null"],"format":"uri"},"client_key":{"type":"string"},"content_hash":{"type":"string"},"due_notified_at":{"type":["string","null"],"format":"date-time"},"updated_at":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/linkedin-post-in-your-voice/post-status@0.1.0","type":"object","required":["post_id","status"],"properties":{"post_id":{"type":"string"},"status":{"enum":["draft","approved","scheduled","published","rejected"]},"approval_id":{"type":"string"},"scheduled_for":{"type":"string","format":"date-time"},"published_url":{"type":"string","format":"uri"}},"allOf":[{"if":{"properties":{"status":{"const":"published"}}},"then":{"required":["published_url"]}},{"if":{"properties":{"status":{"enum":["approved","scheduled","published"]}}},"then":{"required":["approval_id"]}}]}
{"$id":"schema://nova/linkedin-post-in-your-voice/approval@0.1.0","type":"object","required":["approval_id","post_id","action","content_hash","approved_at"],"properties":{"approval_id":{"type":"string"},"post_id":{"type":"string"},"action":{"enum":["approve","approve_and_publish","schedule","publish_now"]},"content_hash":{"type":"string"},"approved_at":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/linkedin-post-in-your-voice/verify-report@0.1.0","type":"object","required":["passed"],"properties":{"passed":{"type":"boolean"},"format_failures":{"type":"array","items":{"enum":["hook_too_long","length_out_of_range","too_many_hashtags","link_in_body","banned_phrase"]}},"claim_failures":{"type":"array","items":{"type":"object","properties":{"text":{"type":"string"},"reason":{"enum":["ungrounded_specific","invented_experience","news_buried"]}}}}}}
{"$id":"schema://nova/linkedin-post-in-your-voice/publish-input@0.1.0","type":"object","required":["post_id","text","content_hash","approval_id","publish_key"],"properties":{"post_id":{"type":"string"},"text":{"type":"string","minLength":1,"maxLength":3000},"content_hash":{"type":"string"},"approval_id":{"type":"string"},"publish_key":{"type":"string"}}}
{"$id":"schema://nova/linkedin-post-in-your-voice/publish-result@0.1.0","type":"object","required":["state"],"properties":{"state":{"enum":["published","unknown","failed","blocked"]},"post_urn":{"type":["string","null"]},"url":{"type":["string","null"],"format":"uri"},"reason":{"type":["string","null"]},"published_at":{"type":["string","null"],"format":"date-time"}}}
{"$id":"schema://nova/linkedin-post-in-your-voice/connection-state@0.1.0","type":"object","required":["state"],"properties":{"state":{"enum":["active","blocked"]},"reason":{"enum":[null,"no_app","not_approved","expired","missing_scope"]},"expires_at":{"type":["string","null"],"format":"date-time"},"posting_scope_present":{"type":"boolean"}}}
{"$id":"schema://nova/linkedin-post-in-your-voice/delivery@0.1.0","type":"object","required":["delivered"],"properties":{"message_id":{"type":["string","null"]},"channel":{"enum":["owner_chat","secondary"]},"delivered":{"type":"boolean"},"delivered_at":{"type":["string","null"],"format":"date-time"},"error":{"type":["string","null"]}}}
```

**Example (fictional)**

Input: `{"topic_id":"tp_0412","angle":"Parallel tools matter less than good tool descriptions.","intent":"draft_only"}`

Output: `{"post_id":"post_0088","status":"completed","queue_status":"draft","platform":"linkedin","hook":"An open-source agent framework just shipped parallel tool calling. Here's what actually changes for builders.","body":"…1,420 characters in the user's voice, facts from the release notes, the user's angle in the closing paragraph…","char_count":1531,"hashtags":["#AIagents"],"suggested_first_comment":"Release notes: https://example.com/news/parallel-tool-calling","image_prompt":"4:5 portrait content card, headline 'Parallel tools, same rules' …","sources":["https://example.com/news/parallel-tool-calling"],"voice_tuned":true,"model_tier":"pro","publish":{"state":"not_requested","url":null,"reason":null}}`

## E. Constraints and budgets

- **Hard constraints:** nothing is published without a per-post approval bound to the exact content hash (any edit after approval voids it); publishing only through the user's own LinkedIn grant — if it is missing, expired or lacks the posting scope, publishing is `blocked`, with no workaround; scheduled posts are never auto-published (the user gets a reminder and approves then); facts only from the source and the user's brief or angle; never invent the user's opinions, anecdotes, clients, results or numbers; for a launch or release, lead with what actually shipped before any opinion; no tagging of people or companies the user did not name; image prompts never depict real third-party logos.
- **Soft constraints (LinkedIn format profile):** hook ≤ 210 characters (visible before "see more"); total 1,300–1,900 characters; 0–3 hashtags; no links in the body (offered as a suggested first comment); none of the user's banned phrases or common clichés; image prompt for a 4:5 portrait card.
- **Resource and tenant scope:** the user's voice profile, topics, queue and own LinkedIn profile.
- **Data sources allowed:** the topic, the fetched source article, the user's brief and angle, the voice profile.
- **External transfers permitted:** source text, brief and voice profile to the selected model provider; the approved post text to LinkedIn.
- **Time limit:** draft ≤ 2 minutes; approval wait ≤ 24 hours (then the draft simply stays in the queue).
- **Cost or token budget:** per-draft token cap `TBD` (operator); at most 1 regeneration without a new user request.
- **Maximum steps:** 11 (S1–S11).
- **Maximum retries and correction cycles:** fetch 1; drafting 1; verification-driven redraft 1; publish **0 automatic retries**; correction cycles 2 per run.
- **Rate limits and concurrency:** publishes per day capped by the operator (`TBD`, proposed 2) and by LinkedIn's limits; one publish in flight per user.
- **Retention and logging constraints:** queue items kept until the user deletes them (`TBD`); logs never contain tokens; post text kept only in the queue.
- **Prohibited actions:** comments, messages, connection requests, reactions; posting to another profile or page; browser automation; re-publishing a published post; changing approved text without a new approval.
- **Stop conditions:** user rejects; tier unavailable; connection blocked (for publish); claim check fails after one redraft.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Post in the queue (`draft`) | S5 passed | Allowed | read back by client key |
| Create | Approval record | user action in S7 or S11 | Allowed (it is the user's decision) | hash matches the post |
| Read | voice profile, topic, source article, own queue | own tenant; public article | Allowed | — |
| Update | Post `hook`, `body`, `image_prompt` | status not `published`; voids prior approval | Allowed on user request | new content hash |
| Update | Post `status` | valid transition; `published` only with a LinkedIn URL | Approval Required for `approved`, `scheduled`, `published` (user action) | status read back |
| Delete | any queue item or LinkedIn post | — | Not Allowed (the user deletes manually) | — |
| External side effect | create a LinkedIn post | approved hash, active grant, daily cap | Approval Required (per post) | post URL returned and recorded |
| External side effect | due reminder to the user | scheduled post due | Allowed (owner only) | delivery receipt |

- **Idempotency:** queue client key = hash(`tenant_id`, source ref, `run_id`); publish key = hash(`post_id`, content hash). A post already `published` is never published again.
- **Expected side effects:** queue rows; a public LinkedIn post after approval; a reminder message for scheduled posts.
- **Compensation:** the agent cannot unpublish; it gives the user the post link to edit or delete it on LinkedIn.
- **Irreversible:** a published post (visible to the user's network immediately).
- **Ambiguous outcomes:** a publish timeout sets `publish.state = unknown`; the agent asks the user to check their profile (or reads recent posts if the API allows) before any retry; never auto-retry.

## G. Autonomy and approval policy

- **Default autonomy level:** A1 — supervised execution.
- **Planning level:** A2.
- **Execution level:** A2 for drafting and queueing; A0 for approve, schedule and publish decisions (the user's own actions).
- **Verification level:** A2.
- **Recovery level:** A1 — no publish retry without the user.
- **Approval required for:** approving a draft; scheduling; publishing (per post, exact text); editing an approved post (re-approval).
- **Actions never permitted:** publishing without approval; auto-publishing scheduled posts; any workaround when the connection is blocked; outreach of any kind; invented facts or experiences.
- **Conditions requiring clarification:** unclear topic or angle; brief that reads like the user's own news (route to the announcement recipe).
- **Conditions requiring escalation:** the source makes claims about a named person or a sensitive event; the connection is blocked; publish outcome unknown.
- **Safe state after stop, cancellation or escalation:** the draft remains in the queue as `draft` or `approved`; nothing published; the user told.

Approval covers one post with one content hash. It does not cover edits, other posts or re-publishing.

## H. Execution plan

**S1 — Validate the request**
1. Objective: a usable topic or brief.
2. Capability: none (local validation); `nova.topics.store.get.default` when `topic_id` is given.
3. Input mapping: user message → `request@0.1.0`.
4. Dependencies: none.
5. Preconditions: an explicit request for a LinkedIn post.
6. Action / side effects: validate; load the topic; no side effects.
7. Autonomy: A2; no approval.
8. Timeout / retry: 10 s / 1 retry for the topic read.
9. Idempotency: read-only.
10. Postconditions: a schema-valid request with source material references.
11. Verification: JSON Schema passes; topic belongs to the tenant.
12. Failure routes: missing input → ask once → `blocked`; the brief is the user's own news → offer the announcement recipe.
13. Escalation triggers: none.
14. Evidence: request (without confidential text), topic ID.

**S2 — Load the voice profile**
1. Objective: the user's style, examples and banned phrases.
2. Capability: `nova.content.voice.get_profile.default`.
3. Input mapping: tenant.
4. Dependencies: S1.
5. Preconditions: none.
6. Action / side effects: read-only.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: guide, 0–5 examples, banned phrases.
11. Verification: guide present.
12. Failure routes: no examples → continue with `voice_tuned = false` and disclose; no profile → use the default guide and disclose.
13. Escalation triggers: none.
14. Evidence: profile version, example count.

**S3 — Read the source**
1. Objective: the facts the post may use.
2. Capability: `nova.research.fetch.get.default`.
3. Input mapping: `source_url` or the topic URL.
4. Dependencies: S1.
5. Preconditions: a public URL.
6. Action / side effects: HTTP GET.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 1 retry.
9. Idempotency: read-only.
10. Postconditions: source text, or "no source" (brief only).
11. Verification: text ≥ 500 characters and on topic.
12. Failure routes: unreadable → draft from the topic title, snippet and brief only; facts limited accordingly; disclose.
13. Escalation triggers: none.
14. Evidence: URL, status, content hash.

**S4 — Draft the post**
1. Objective: a LinkedIn post in the user's voice.
2. Capability: `nova.content.draft.write.<model_tier>`.
3. Input mapping: voice profile, source text (as data), topic, brief, angle, LinkedIn format profile, `include_image_prompt`.
4. Dependencies: S2, S3.
5. Preconditions: selected tier ready.
6. Action / side effects: inputs sent to the model provider; output = hook, body, hashtags, suggested first comment and image prompt, split by the delimiter line.
7. Autonomy: A2.
8. Timeout / retry: 90 s / 1 retry on transient errors; no tier switch.
9. Idempotency: not applicable.
10. Postconditions: a parsed draft.
11. Verification: deferred to S5; an empty completion is a failure.
12. Failure routes: tier unavailable → `failed` with the reason.
13. Escalation triggers: none.
14. Evidence: tier and model ID, token counts.

**S5 — Verify the draft**
1. Objective: a draft that follows the rules and states only supported facts.
2. Capability: `nova.content.verify.format.linkedin`; `nova.content.verify.claims.rules` (plus `llm-check` when enabled).
3. Input mapping: draft, source text, brief, angle, banned phrases.
4. Dependencies: S4.
5. Preconditions: draft parsed.
6. Action / side effects: check lengths, hashtags, links in body, banned phrases; check every number, name, date and product in the draft appears in the source, brief or angle; check no first-person experience is asserted that the user did not supply; for a launch, check the news appears in the hook or first two sentences.
7. Autonomy: A2.
8. Timeout / retry: 15 s / none.
9. Idempotency: deterministic.
10. Postconditions: pass, or a list of failures.
11. Verification: 100 % of checks pass.
12. Failure routes: failures → redraft once naming them → still failing → `blocked` with the failures shown (nothing queued).
13. Escalation triggers: claims about a named person or a sensitive event → show the user before queueing.
14. Evidence: check results.

**S6 — Save to the approval queue**
1. Objective: the draft is safe in the queue.
2. Capability: `nova.content.queue.save_draft.default`.
3. Input mapping: draft, source refs, client key, content hash.
4. Dependencies: S5.
5. Preconditions: S5 passed.
6. Action / side effects: a `draft` row in the user's queue.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries after reconciling by client key.
9. Idempotency: client key.
10. Postconditions: post readable with status `draft`.
11. Verification: read back; content hash matches.
12. Failure routes: save fails → show the draft to the user anyway and mark `partially_completed`.
13. Escalation triggers: none.
14. Evidence: post ID.

**S7 — User review**
1. Objective: the user decides.
2. Capability: user interaction; `nova.content.queue.edit` / `set_status`.
3. Input mapping: draft, sources, check summary, `voice_tuned`, connection state.
4. Dependencies: S6.
5. Preconditions: draft saved.
6. Action / side effects: present Approve · Approve and publish now · Schedule · Edit · Regenerate · Reject. Approve and Schedule update status; Edit saves a new version (back to S5); Regenerate returns to S4 once; Reject sets `rejected`.
7. Autonomy: A0 — the user's decision.
8. Timeout / retry: no timeout on the queue item; the run ends after 24 hours with the draft kept.
9. Idempotency: each approval record binds one content hash.
10. Postconditions: an approval record, or the draft left as is.
11. Verification: approval hash = current content hash.
12. Failure routes: no response → `completed` for `draft_only`, `partially_completed` for `draft_and_publish`.
13. Escalation triggers: none.
14. Evidence: approval ID, action, hash.

**S8 — Check the LinkedIn connection** (only after "Approve and publish now", or at a due reminder)
1. Objective: know whether publishing is possible.
2. Capability: `nova.content.publish.check_connection.default`.
3. Input mapping: tenant's LinkedIn grant reference.
4. Dependencies: S7 (or S11).
5. Preconditions: a publish approval exists.
6. Action / side effects: read-only check of grant validity, expiry and posting scope.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 1 retry.
9. Idempotency: read-only.
10. Postconditions: `active`, or `blocked` with a reason (`no_app`, `not_approved`, `expired`, `missing_scope`).
11. Verification: expiry later than now plus 5 minutes.
12. Failure routes: blocked → stop; post stays `approved`; tell the user how to reconnect and offer the text for manual copy-paste.
13. Escalation triggers: blocked connection.
14. Evidence: connection state and reason (no token).

**S9 — Publish** (approval required)
1. Objective: exactly one LinkedIn post with the approved text.
2. Capability: `nova.content.publish.linkedin.member-post`.
3. Input mapping: approved hook + body (+ hashtags), publish key.
4. Dependencies: S8.
5. Preconditions: connection `active`; approval hash = current hash; status not `published`; daily cap not reached.
6. Action / side effects: a public post on the user's profile.
7. Autonomy: A1, covered by the per-post approval.
8. Timeout / retry: 20 s / **no automatic retry**.
9. Idempotency: publish key; a published post is never re-sent.
10. Postconditions: a post URL/URN returned.
11. Verification: response contains a post identifier.
12. Failure routes: rejection → `failed` with LinkedIn's reason; timeout → `publish.state = unknown` → S10 reconciliation.
13. Escalation triggers: unknown outcome.
14. Evidence: post URN, timestamp.

**S10 — Confirm and record**
1. Objective: the queue reflects reality.
2. Capability: `nova.content.queue.set_status.default`.
3. Input mapping: post ID, URL/URN or the unknown state.
4. Dependencies: S9.
5. Preconditions: a publish attempt was made.
6. Action / side effects: set `published` with the URL only when an identifier was returned; otherwise leave `approved` with `publish.state = unknown` and ask the user to check their profile.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: same status twice is a no-op.
10. Postconditions: status and URL consistent.
11. Verification: read back.
12. Failure routes: status write fails → report the URL to the user and retry the write; `partially_completed`.
13. Escalation triggers: unknown outcome unresolved.
14. Evidence: status, URL.

**S11 — Due reminder for scheduled posts** (separate periodic check, every 5 minutes)
1. Objective: the user is reminded when a scheduled post is due; nothing is auto-published.
2. Capability: `nova.content.due.notify.default`.
3. Input mapping: posts with `status = scheduled` and `scheduled_for ≤ now` not yet notified for that time.
4. Dependencies: S7 (Schedule).
5. Preconditions: owner channel configured.
6. Action / side effects: a reminder with the post and a "Publish now" action (which goes to S8–S10 with a new per-post approval).
7. Autonomy: A2 for the reminder; A0 for publishing.
8. Timeout / retry: 15 s / retried at the next tick.
9. Idempotency: send first, then mark notified, so an outage retries next tick and never drops a reminder; rescheduling re-arms it.
10. Postconditions: reminder delivered and marked.
11. Verification: delivery receipt before marking.
12. Failure routes: delivery fails → not marked; retried next tick.
13. Escalation triggers: reminder undelivered for 1 hour → note in the run record.
14. Evidence: message ID, notified timestamp.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Input validity | request matches schema; topic in tenant | validator | valid | ask once, then `blocked` |
| Format | LinkedIn profile rules met | S5 format report | all pass | redraft once |
| Claim grounding | every specific traces to source, brief or angle | S5 claims report | 100 % | redraft once, then `blocked` |
| No invented experience | no first-person anecdote or result the user did not supply | S5 | none found | redraft |
| Approval binding | approval hash = published text hash | approval vs publish payload | equal | stop before publishing |
| Connection | grant active with posting scope | S8 | active | `blocked` |
| Publish truth | `published` only with a returned post identifier | S9/S10 | true | set `unknown`, reconcile |
| No auto-publish | scheduled posts only produce reminders | S11 log | true | stop, escalate |
| Output schema | `post_result` valid | validator | valid | fix fields; never fake values |

A 200 response from the publish endpoint without a post identifier is not proof of publication.

## J. Self-correction and recovery

- **Failure classes:** validation, unreadable source, tier unavailable, empty completion, format failure, ungrounded claim, queue timeout, connection blocked, publish rejected, publish outcome unknown, reminder delivery failure.
- **Permitted corrections:** ask once; redraft once with the failures named; reconcile a queue save by client key; retry reminders next tick.
- **Forbidden corrections:** publishing on any failure; retrying a publish with an unknown outcome; any workaround for a blocked connection; switching tier silently; loosening claim rules.
- **Maximum correction cycles:** 1 per step; 2 per run.
- **Reverification required after correction:** yes — S5 re-runs on every new version.
- **Stop conditions:** user rejection; connection blocked; correction budget exhausted.
- **Recovery or compensation:** give the user the post link to edit or delete on LinkedIn.
- **What must be reported:** redrafts, voice not tuned, missing source, blocked connection with the fix, unknown publish state.

## K. Independent challenge

- **Assumptions to challenge:** the draft really sounds like the user (compare with the examples); the source supports the headline claim; the user's angle is represented faithfully.
- **Alternative approach:** if the source is thin, propose a shorter opinion post that relies only on the user's angle.
- **Contradictory evidence:** other coverage that disputes the source (offer a web-research check before publishing).
- **Could mislead:** a confident framing of an unconfirmed report; a hook that promises more than the body delivers.
- **Separate verifier:** S5 rules are independent of the writer; the optional LLM check runs on a different model.
- **Findings requiring disclosure:** voice not tuned, single-source facts, publish blocked.

## L. Human escalation

- **Triggers:** connection blocked; publish outcome unknown; claims about a named person or a sensitive event; claim check failing after a redraft.
- **Why the agent cannot decide:** public statements under the user's name; account access belongs to the user.
- **Information presented:** the draft, sources, failed checks, connection state and how to fix it.
- **Decision requested:** approve, edit, reject; reconnect LinkedIn; confirm whether a post appeared.
- **Options and consequences:** publishing is visible immediately; manual copy-paste bypasses nothing and keeps the user in control.
- **Safe behaviour while waiting:** the draft stays in the queue; nothing published.
- **If no response:** the draft remains `draft` or `approved` indefinitely.
- **Resume:** S8 re-checks the connection and S9 requires a fresh approval if the text changed.

## M. Completion conditions

Mandatory: `post_result` valid; draft passed S5; queue state matches reality; published only with approval and a returned identifier; limitations disclosed; evidence references present.

- `completed` — `draft_only`: draft queued (and approved or scheduled if the user chose); `draft_and_publish`: post published with a URL.
- `partially_completed` — draft queued but the queue save or a status write failed, or publish requested and the user did not respond.
- `blocked` — invalid input, claim checks failing, or publishing blocked by the LinkedIn connection (draft stays queued).
- `failed` — tier unavailable, LinkedIn rejected the post, or the publish outcome remained unknown and unreconciled.
- `cancelled` — the user rejected the draft or cancelled.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome ("draft queued — publishing blocked: LinkedIn connection expired"); `post_result`; step statuses S1–S11; verification results and failed checks; evidence references (post ID, approval ID and hash, source URLs, post URN); approvals and scope; side effects (queue rows, published post) and reconciliation status; corrections performed; unresolved challenges; limitations; escalation state; learning candidate. Never include OAuth tokens, app secrets or confidential brief text in shared channels.

## O. Learning and revision

- **Outcome metrics:** approval rate, edit distance between draft and approved text, regeneration rate, rejection reasons, claim-check failures, publish success and blocked rates.
- **Evidence retained:** queue items and approval records, per the retention policy.
- **Data minimisation:** metrics without post text; edit distance as a number only.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}` — for example, adding a phrase the user always deletes to the banned list (proposed, not applied, until the user agrees).
- **Review owner:** the publisher (TBD); voice-profile changes by the user.
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review; voice-profile changes need the user's consent.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path, draft only (fictional topic) | draft queued, checks pass | UNVERIFIED on Skill Port; observed in the reference implementation |
| Happy path, publish (LinkedIn test account) | one post, URL recorded | TBD (blocked until an approved app and grant exist) |
| Missing topic and brief | asks once, then `blocked` | TBD |
| Invalid input (past `schedule_for`) | rejected | TBD |
| Missing capability (no LinkedIn app) | publish `blocked`, draft kept | TBD |
| Permission denied (expired grant or missing posting scope) | `blocked` with the reconnect instructions | TBD |
| Unauthorized scope (another tenant's topic or post) | denied | TBD |
| Tenant isolation | queue reads only the user's posts | TBD |
| Publish timeout | `unknown`; no retry; user asked | TBD |
| Duplicate / replay publish | already `published` → refused | TBD |
| False success (200 without a post identifier) | not marked published | TBD |
| Ungrounded number or invented anecdote | caught by S5 | TBD |
| Edited after approval | approval void; re-approval required | TBD |
| Correction budget exhausted | `blocked` with failures shown | TBD |
| Escalation (claims about a named person) | shown to the user before queueing | TBD |
| Cancellation (user rejects) | `rejected`, nothing published | TBD |
| Scheduled post due | reminder only, never auto-publish; outage → retried next tick | TBD |
| Compensation (user wants a post removed) | link provided; agent does not delete | TBD |
| Output schema and status | valid `post_result`; correct status | TBD |
| Compatibility / regression (LinkedIn API version change) | detected as publish failure; no partial state | TBD |

All publish tests use a test LinkedIn account owned by the tester; never a real audience. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Publishing is blocked until the user's LinkedIn app has member-posting access and an active grant; long-lived (refreshable) access needs LinkedIn's additional app approval (owner: the user; guidance: publisher).
- The exact LinkedIn permission names and API version must be confirmed against LinkedIn's current documentation before activation (owner: publisher).
- Skill Port / Vedaxi binding IDs, approval UI and schema registry — TBD (owner: publisher, with Skill Port).
- Voice quality depends on the user's example posts; without them drafts are flagged "voice not tuned".
- Daily publish cap and queue retention — TBD (owner: operator).
- Image generation is a separate capability, not part of this recipe.
