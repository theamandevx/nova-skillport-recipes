# Announcement post

## A. Identity and ownership

- **Recipe ID:** `nova.content.announcement-post` (proposed until assigned by Skill Port)
- **Name:** Announcement post
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** marketing
- **Tags:** announcement, launch, milestone, lead-with-the-news, linkedin, x, writing, approval-queue, no-invented-claims · Vedaxi task type: none of the seven supported today maps directly (closest: Save a note draft)
- **Summary:** The user shares their own news — a launch, rebrand, milestone or update — as a rough note. The agent turns it into an announcement post that states the news in its first line, uses only facts from the user's brief, and saves the draft to the approval queue. It never invents claims, numbers, customers, partners or quotes.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** A voice profile; a cheap-tier model for fact extraction and a pro-tier model for drafting (medium variants optional); platform format profiles for LinkedIn and X; the content approval queue. Publishing is out of scope (handled by a separate, per-post-approved publish flow). All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — the announcement drafting mode (frames, a rough brief polished into a post, facts only from the brief, first-person builder voice, LinkedIn and X formats) and the approval queue run in Nova's reference implementation; a test draft was judged faithful to its brief. Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.content.voice` | Capability | Hold the user's writing voice and brand colours | low | UNVERIFIED |
| `nova.content.voice.get_profile` | Operation (read) | Return style guide, examples, banned phrases, brand colours | low | UNVERIFIED |
| `nova.content.facts` | Capability | Turn a rough brief into a checkable fact sheet | low | UNVERIFIED |
| `nova.content.facts.extract` | Operation (read, LLM) | List atomic facts, each with its verbatim span from the brief | low | UNVERIFIED |
| `nova.content.draft` | Capability | Draft platform posts | medium | UNVERIFIED |
| `nova.content.draft.announce` | Operation (read, LLM) | Draft an announcement that leads with the news, using only fact-sheet items | medium | UNVERIFIED |
| `nova.content.verify` | Capability | Check drafts before they reach the queue | low | UNVERIFIED |
| `nova.content.verify.format` | Operation (read) | Apply the platform format profile | low | UNVERIFIED |
| `nova.content.verify.claims` | Operation (read) | Map every claim to a fact-sheet item; check lead-with-news, superlatives and third-party mentions | low | UNVERIFIED |
| `nova.content.queue` | Capability | The user's content approval queue | medium | UNVERIFIED |
| `nova.content.queue.save_draft` | Operation (create) | Save a draft, idempotent on a client key | low | UNVERIFIED |
| `nova.content.queue.edit` | Operation (update) | Change a non-published draft | low | UNVERIFIED |
| `nova.content.queue.set_status` | Operation (update) | `approved` or `rejected` on the user's action | medium | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.content.facts.extract.cheap` — default; extraction is a narrow task. `nova.content.facts.extract.medium` — explicit choice only, for long or messy briefs.
- `nova.content.draft.announce.pro` — default (the reference implementation drafts on its top tier); reasoning-capable; output-token headroom (about 4,000); an empty completion is a failure. `nova.content.draft.announce.medium` — explicit choice only. The cheap tier is not declared for public-facing writing. **No automatic tier switch** for either operation; an unavailable tier fails the step with its reason.
- `nova.content.verify.format.linkedin` — announcement profile: 900–1,900 characters (900–1,400 is acceptable for a thin brief), hook ≤ 210 characters, 0–3 hashtags, no link in the body (link offered for the first comment). `nova.content.verify.format.x` — ≤ 270 characters, one idea, no hashtags, no links (link-detection also catches bare domains such as `example.com` without flagging version numbers like "3.5").
- `nova.content.verify.claims.rules` — default; deterministic. `nova.content.verify.claims.llm-check` — optional, on a model different from the drafter; can only flag.
- Draft output format: caption and image prompt separated by a fixed delimiter line (not JSON inside text).
- All queue operations: only the default variant is supported.

**Bindings:** `binding://nova/content/voice`, `binding://nova/content/facts`, `binding://nova/content/draft-announce`, `binding://nova/content/verify-format`, `binding://nova/content/verify-claims`, `binding://nova/content/queue-save`, `binding://nova/content/queue-edit`, `binding://nova/content/queue-status` — all `status: UNVERIFIED`. Reference implementation: a dashboard "new post" dialog (frame chips, title, rough message, optional link, platform select) → a queue row marked as a custom announcement → the draft gateway (n8n, header-authenticated) running its announcement mode → the content schema in PostgreSQL behind a security-definer dispatcher.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Announcement brief | `schema://nova/announcement-post/request@0.1.0` | the user | read |
| Fact sheet | `schema://nova/announcement-post/fact-sheet@0.1.0` | the run and the queued post | create |
| Voice profile | `schema://nova/announcement-post/voice-profile@0.1.0` | the user | read |
| Post | `schema://nova/announcement-post/post@0.1.0` | the user's tenant | create, read, update (content before publish; status) |
| Mention confirmation | `schema://nova/announcement-post/mention-confirmation@0.1.0` | linked to one Post | create |

**Authorization requirements (declared, never a grant):** `content:voice:read`, `llm:invoke:cheap`, `llm:invoke:pro` or `llm:invoke:medium`, `content:queue:write`, `content:queue:read`. Model keys as `secret_ref://llm-cheap`, `secret_ref://llm-pro`, `secret_ref://llm-medium`. No publishing scope is requested by this recipe.

**Policy / trust:** risk class `medium` — announcements become public statements about the user's business once published (elsewhere, with approval); inventing a claim could mislead customers or breach agreements. The brief is the user's own data; it is still checked, never embellished.

## C. Intent and applicability

- **User problem:** The user has news to share but struggles to turn rough notes into a clear post — and generic AI writing tends to bury the news and add hype the user never claimed.
- **Desired outcome:** A queued draft that says what is new in its first line, in the user's voice, with every claim traceable to the user's own words.
- **Trigger or entry conditions:** The user says they have something to announce and provides a brief (rough is fine).
- **Applicable situations:** Product launches, releases, rebrands, work-in-progress updates, milestones, hiring, events the user is running.
- **Out-of-scope situations:** Someone else's news or industry commentary (use `nova.content.linkedin-post-in-your-voice`); press releases needing legal review; financial results subject to disclosure rules; announcing partners, customers or investors who have not agreed; publishing (separate per-post-approved flow).
- **Required inputs:** `frame`, `brief`, `ownership_confirmed` (see Section D).
- **Prerequisites:** a voice profile (brand colours optional); ready extraction and drafting tiers; the content queue.
- **Required capabilities:** `nova.content.voice`, `nova.content.facts`, `nova.content.draft`, `nova.content.verify`, `nova.content.queue` (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** Publishing and image generation are not part of this recipe.
- **Assumptions:** Verified in the reference implementation: announcement mode, LinkedIn and X formats, queueing. Assumed: the user is entitled to announce what they submit (confirmed per request).
- **Ambiguity policy:** Ask when the news itself is unclear (what changed, for whom, since when); never fill gaps with invented specifics — a thinner post is the safe default.
- **Vedaxi task type:** None maps directly; the closest supported type is Save a note draft.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `frame` | enum `launch`, `rebrand`, `building`, `milestone`, `announcement` | yes | user | `announcement` | Unclear: ask once. |
| `platform` | enum `linkedin`, `x` | no | user | `linkedin` | Other platforms: not supported in this version. |
| `title` | string ≤ 120 chars | no | user | derived | — |
| `brief` | string, 20–3,000 chars | yes | user; may be confidential until announced | — | Too thin to state what changed: ask once. |
| `link` | URI (`http`/`https`) | no | user | none | Invalid: drop and ask. |
| `ownership_confirmed` | boolean, must be `true` | yes | user | — | `false`/missing: ask; still not confirmed → `blocked`. |
| `embargo_until` | date-time with offset | no | user | none | In the past: ignored with a note. |
| `model_tier` | enum `medium`, `pro` | no | user / operator | `pro` | Not ready: fail with reason; no switch. |
| `include_image_prompt` | boolean | no | user | `true` | — |

```json
{"$id":"schema://nova/announcement-post/request@0.1.0","type":"object","required":["frame","brief","ownership_confirmed"],
 "properties":{"frame":{"enum":["launch","rebrand","building","milestone","announcement"]},"platform":{"enum":["linkedin","x"]},
 "title":{"type":"string","maxLength":120},"brief":{"type":"string","minLength":20,"maxLength":3000},
 "link":{"type":"string","format":"uri","pattern":"^https?://"},"ownership_confirmed":{"const":true},
 "embargo_until":{"type":"string","format":"date-time"},"model_tier":{"enum":["medium","pro"]},"include_image_prompt":{"type":"boolean"}}}
```

```json
{"$id":"schema://nova/announcement-post/fact-sheet@0.1.0","type":"object","required":["news","facts"],
 "properties":{"news":{"type":"string","maxLength":200,"description":"one sentence: what is new"},
 "facts":{"type":"array","minItems":1,"items":{"type":"object","required":["id","statement","span"],
  "properties":{"id":{"type":"string"},"statement":{"type":"string"},"span":{"type":"string","description":"verbatim text from the brief"},
  "kind":{"enum":["what","who","when","number","price","link","third_party","other"]}}}}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `announcement_result` | `schema://nova/announcement-post/announcement-result@0.1.0` | `post_id`, `status`, `queue_status`, `platform`, `text`, `claim_map` | the user's conversation; the queue; run record | confidential until published | S5 checks; claim map shown at review |

```json
{"$id":"schema://nova/announcement-post/announcement-result@0.1.0","type":"object",
 "required":["post_id","status","queue_status","platform","text","claim_map"],
 "properties":{"post_id":{"type":"string"},"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},
 "queue_status":{"enum":["draft","approved","rejected"]},"platform":{"enum":["linkedin","x"]},
 "hook":{"type":"string","maxLength":210},"text":{"type":"string"},"char_count":{"type":"integer"},
 "suggested_first_comment":{"type":["string","null"]},"image_prompt":{"type":["string","null"]},
 "claim_map":{"type":"array","items":{"type":"object","required":["claim","fact_ids"],
  "properties":{"claim":{"type":"string"},"fact_ids":{"type":"array","minItems":1,"items":{"type":"string"}}}}},
 "third_party_mentions":{"type":"array","items":{"type":"object","properties":{"name":{"type":"string"},"confirmed":{"type":"boolean"}}}},
 "embargo_until":{"type":["string","null"],"format":"date-time"},"model_tier":{"enum":["medium","pro"]}}}
```

Supporting schemas (compact, one per line):

```json
{"$id":"schema://nova/announcement-post/voice-query@0.1.0","type":"object","required":["tenant_id"],"properties":{"tenant_id":{"type":"string"},"platform":{"enum":["linkedin","x"]}}}
{"$id":"schema://nova/announcement-post/voice-profile@0.1.0","type":"object","required":["guide"],"properties":{"guide":{"type":"string","minLength":1},"examples":{"type":"array","maxItems":5,"items":{"type":"string"}},"banned_phrases":{"type":"array","items":{"type":"string"}},"brand_colours":{"type":"array","items":{"type":"string","pattern":"^#[0-9A-Fa-f]{6}$"}},"version":{"type":"string"}}}
{"$id":"schema://nova/announcement-post/post@0.1.0","type":"object","required":["post_id","platform","body","status","content_hash"],"properties":{"post_id":{"type":"string"},"tenant_id":{"type":"string"},"platform":{"enum":["linkedin","x"]},"hook":{"type":"string","maxLength":210},"body":{"type":"string"},"image_prompt":{"type":["string","null"]},"source_type":{"const":"announcement"},"frame":{"enum":["launch","rebrand","building","milestone","announcement"]},"fact_sheet":{"$ref":"schema://nova/announcement-post/fact-sheet@0.1.0"},"embargo_until":{"type":["string","null"],"format":"date-time"},"status":{"enum":["draft","approved","rejected"]},"client_key":{"type":"string"},"content_hash":{"type":"string"}}}
{"$id":"schema://nova/announcement-post/post-status@0.1.0","type":"object","required":["post_id","status"],"properties":{"post_id":{"type":"string"},"status":{"enum":["approved","rejected"]},"approval_id":{"type":"string"}},"allOf":[{"if":{"properties":{"status":{"const":"approved"}}},"then":{"required":["approval_id"]}}]}
{"$id":"schema://nova/announcement-post/verify-report@0.1.0","type":"object","required":["passed"],"properties":{"passed":{"type":"boolean"},"format_failures":{"type":"array","items":{"type":"string"}},"claim_failures":{"type":"array","items":{"type":"object","required":["claim","reason"],"properties":{"claim":{"type":"string"},"reason":{"enum":["unmapped","ungrounded_specific","unbacked_superlative","news_buried","unconfirmed_mention"]}}}}}}
{"$id":"schema://nova/announcement-post/mention-confirmation@0.1.0","type":"object","required":["post_id","name","confirmed"],"properties":{"post_id":{"type":"string"},"name":{"type":"string"},"confirmed":{"type":"boolean"},"confirmed_at":{"type":["string","null"],"format":"date-time"}}}
```

**Example (fictional)**

Input: `{"frame":"launch","platform":"linkedin","brief":"we launched example notes 2.0 today. offline mode, shared notebooks, export to markdown. free for individuals, teams $6/user/month. took us 5 months, 3 of us.","link":"https://example.com/notes-2","ownership_confirmed":true}`

Output: `{"post_id":"post_0091","status":"completed","queue_status":"draft","platform":"linkedin","hook":"Example Notes 2.0 is live today: offline mode, shared notebooks and Markdown export.","text":"Example Notes 2.0 is live today: offline mode, shared notebooks and Markdown export.\n\nThree of us spent five months on it…\n\nFree for individuals. Teams: $6 per user per month.\n\nLink in the comments.","char_count":1012,"suggested_first_comment":"https://example.com/notes-2","image_prompt":"4:5 portrait card in the user's brand colours, headline 'Example Notes 2.0 is live' …","claim_map":[{"claim":"Example Notes 2.0 is live today","fact_ids":["f1"]},{"claim":"offline mode, shared notebooks and Markdown export","fact_ids":["f2","f3","f4"]},{"claim":"Three of us spent five months on it","fact_ids":["f7","f8"]},{"claim":"Free for individuals. Teams: $6 per user per month.","fact_ids":["f5","f6"]}],"third_party_mentions":[],"embargo_until":null,"model_tier":"pro"}`

## E. Constraints and budgets

- **Hard constraints:** the user's own news only, with `ownership_confirmed = true`; every factual claim in the draft maps to a fact-sheet item whose span appears verbatim in the brief; no numbers, prices, dates, customers, partners, investors, awards, quotes or rankings that are not in the brief; no superlatives ("first", "only", "leading", "#1", "best") unless the brief states them; the news is stated in the hook (first line); third-party names appear only after the user confirms each mention is public and agreed; nothing is published by this recipe; an embargoed draft cannot be marked ready-to-publish before `embargo_until`; image prompts use the user's brand colours and never depict real third-party logos.
- **Soft constraints:** first-person builder voice ("we shipped" rather than "excited to share"); keep the user's own phrasing when the brief is already a draft; CTA "link in the comments" only when a link exists; format profile limits (Section B).
- **Resource and tenant scope:** the user's voice profile and queue.
- **Data sources allowed:** the brief and the voice profile only — no web lookups (the user's news may not be public yet).
- **External transfers permitted:** the brief and voice profile to the selected model providers.
- **Time limit:** ≤ 3 minutes to a queued draft.
- **Cost or token budget:** per-draft token cap `TBD` (operator); at most 1 automatic redraft.
- **Maximum steps:** 9 (S1–S9).
- **Maximum retries and correction cycles:** extraction 1; drafting 1; verification-driven redraft 1; correction cycles 2 per run.
- **Rate limits and concurrency:** one announcement draft in progress per user.
- **Retention and logging constraints:** brief and fact sheet stored with the queued post only; logs hold IDs and check results, not the brief.
- **Prohibited actions:** web research to "enrich" the announcement; inventing metrics, testimonials or partners; publishing; implying endorsement by a third party.
- **Stop conditions:** ownership not confirmed; claims failing after one redraft; tier unavailable; unconfirmed third-party mentions the user will not resolve.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Post in the queue (`draft`) with its fact sheet | S5 passed; S6 resolved | Allowed | read back by client key |
| Create | Mention confirmation | user confirms a third-party name | Allowed (the user's decision) | record linked to the post |
| Read | voice profile; own queue | own tenant | Allowed | — |
| Update | Post `hook`, `body`, `image_prompt` | not published; voids approval | Allowed on user request | new content hash; S5 re-run |
| Update | Post `status` → `approved` / `rejected` | user action; embargo respected for readiness | Approval Required (user action) | status read back |
| Delete | any | — | Not Allowed | — |
| External side effect | none (no publishing, no messages to third parties) | — | Not applicable | — |

- **Idempotency:** client key = hash(`tenant_id`, brief hash, `platform`); the same brief and platform return the existing draft unless the user asks for a new version.
- **Expected side effects:** one queue row per platform draft.
- **Compensation:** none needed; a draft can be rejected.
- **Irreversible:** nothing in this recipe (publishing happens elsewhere, with its own approval).
- **Ambiguous outcomes:** save timeout → read by client key before retrying.

## G. Autonomy and approval policy

- **Default autonomy level:** A1 — supervised execution.
- **Planning level:** A2.
- **Execution level:** A2 for extraction, drafting and queueing; A0 for approval and mention confirmations.
- **Verification level:** A2.
- **Recovery level:** A2 within limits.
- **Approval required for:** marking a draft `approved`; every third-party mention; overriding the embargo (not allowed — the user changes the embargo instead).
- **Actions never permitted:** inventing claims; publishing; web lookups; announcing third parties without confirmation.
- **Conditions requiring clarification:** unclear news; missing ownership confirmation; a brief that describes someone else's news.
- **Conditions requiring escalation:** brief mentions regulated topics (financial results, health claims, security incidents); unconfirmed third-party names.
- **Safe state after stop, cancellation or escalation:** nothing queued that failed checks; the brief kept only in the run until the user decides.

## H. Execution plan

**S1 — Validate the brief**
1. Objective: a usable brief about the user's own news.
2. Capability: none (local validation).
3. Input mapping: user message → `request@0.1.0`.
4. Dependencies: none.
5. Preconditions: an explicit announcement request.
6. Action / side effects: validate; check `ownership_confirmed`; detect briefs about another organisation's news; no side effects.
7. Autonomy: A2.
8. Timeout / retry: 5 s / none.
9. Idempotency: not applicable.
10. Postconditions: schema-valid request.
11. Verification: JSON Schema passes.
12. Failure routes: ownership not confirmed → ask once → `blocked`; someone else's news → offer the commentary recipe and stop.
13. Escalation triggers: regulated topic detected → S6.
14. Evidence: request hash (no brief text in logs).

**S2 — Build the fact sheet**
1. Objective: the complete list of claims the post may make.
2. Capability: `nova.content.facts.extract.cheap` (or `.medium` if chosen).
3. Input mapping: brief, title, link, frame.
4. Dependencies: S1.
5. Preconditions: tier ready.
6. Action / side effects: brief sent to the model provider; returns one-sentence `news` plus atomic facts, each with a verbatim span.
7. Autonomy: A2.
8. Timeout / retry: 30 s / 1 retry; no tier switch.
9. Idempotency: not applicable.
10. Postconditions: `fact-sheet@0.1.0`.
11. Verification: every span occurs verbatim in the brief; `news` maps to at least one fact.
12. Failure routes: non-verbatim span → drop that fact; no facts left → ask the user what is new.
13. Escalation triggers: `third_party` facts → S6 later.
14. Evidence: fact count, dropped facts.

**S3 — Load the voice profile**
1. Objective: style, banned phrases, brand colours.
2. Capability: `nova.content.voice.get_profile.default`.
3. Input mapping: tenant.
4. Dependencies: S1.
5. Preconditions: none.
6. Action / side effects: read-only.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: profile loaded (default guide if none; disclosed).
11. Verification: guide present.
12. Failure routes: none blocking.
13. Escalation triggers: none.
14. Evidence: profile version.

**S4 — Draft the announcement**
1. Objective: a post that leads with the news.
2. Capability: `nova.content.draft.announce.<model_tier>`.
3. Input mapping: fact sheet (the only allowed facts), the brief as the base text, frame, platform profile, voice profile, link, `include_image_prompt`.
4. Dependencies: S2, S3.
5. Preconditions: tier ready.
6. Action / side effects: inputs sent to the model provider; hook states the news; body adds fact-sheet details; CTA when a link exists; image prompt in brand colours.
7. Autonomy: A2.
8. Timeout / retry: 90 s / 1 retry; no tier switch.
9. Idempotency: not applicable.
10. Postconditions: parsed draft with a claim map (claim → fact IDs).
11. Verification: deferred to S5; an empty completion is a failure.
12. Failure routes: tier unavailable → `failed`.
13. Escalation triggers: none.
14. Evidence: tier and model ID.

**S5 — Verify the draft**
1. Objective: nothing invented, news first, format met.
2. Capability: `nova.content.verify.format.<platform>`; `nova.content.verify.claims.rules` (plus `llm-check` when enabled).
3. Input mapping: draft, claim map, fact sheet, brief.
4. Dependencies: S4.
5. Preconditions: draft parsed.
6. Action / side effects: every sentence with a factual claim maps to ≥ 1 fact; every number, name, date and price appears in the brief; no superlative absent from the brief; the hook contains the `news` subject; banned phrases absent; platform limits met; X drafts contain no link or hashtag.
7. Autonomy: A2.
8. Timeout / retry: 15 s / none.
9. Idempotency: deterministic.
10. Postconditions: pass, or failures listed.
11. Verification: 100 % pass.
12. Failure routes: failures → redraft once naming them → still failing → `blocked` with failures shown; nothing queued.
13. Escalation triggers: none.
14. Evidence: per-check results, claim map.

**S6 — Confirm third-party mentions and sensitive topics** (only when present)
1. Objective: no third party is named and no regulated claim is made without the user's explicit confirmation.
2. Capability: user interaction.
3. Input mapping: third-party facts; regulated-topic flags.
4. Dependencies: S5.
5. Preconditions: at least one mention or flag.
6. Action / side effects: ask per name "Is this public and agreed with them?"; for regulated topics, recommend the user's own review.
7. Autonomy: A0.
8. Timeout / retry: wait up to 24 hours, then the run ends with nothing queued.
9. Idempotency: one confirmation record per name per post.
10. Postconditions: each mention confirmed, or removed from the draft (then S5 re-runs).
11. Verification: no unconfirmed name remains.
12. Failure routes: user declines → remove the mention and redraft once.
13. Escalation triggers: as above.
14. Evidence: confirmation records.

**S7 — Save to the approval queue**
1. Objective: the draft is safe in the queue with its fact sheet.
2. Capability: `nova.content.queue.save_draft.default`.
3. Input mapping: draft, fact sheet, frame, platform, embargo, client key.
4. Dependencies: S5 (and S6).
5. Preconditions: checks passed; mentions resolved.
6. Action / side effects: a `draft` row in the user's queue.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries after reconciling by key.
9. Idempotency: client key.
10. Postconditions: post readable.
11. Verification: read back; content hash matches.
12. Failure routes: save fails → show the draft to the user; `partially_completed`.
13. Escalation triggers: none.
14. Evidence: post ID.

**S8 — User review**
1. Objective: the user decides.
2. Capability: user interaction; `nova.content.queue.edit` / `set_status`.
3. Input mapping: draft, claim map (each claim beside its brief span), check summary, embargo.
4. Dependencies: S7.
5. Preconditions: draft saved.
6. Action / side effects: Approve · Edit · Regenerate · Reject. Edits re-run S5. Approval before `embargo_until` is recorded as approved-with-embargo; it does not make the post ready to publish until then.
7. Autonomy: A0.
8. Timeout / retry: none on the queue item; the run ends after 24 hours with the draft kept.
9. Idempotency: approval bound to the content hash.
10. Postconditions: status `approved`, `rejected`, or `draft`.
11. Verification: approval hash = content hash.
12. Failure routes: no response → `completed` (draft queued).
13. Escalation triggers: none.
14. Evidence: approval ID and hash.

**S9 — Report the result**
1. Objective: tell the user what was produced and what happens next.
2. Capability: none (conversation reply).
3. Input mapping: `announcement_result`.
4. Dependencies: S8 (or S7).
5. Preconditions: result schema-valid.
6. Action / side effects: reply with the draft, the claim map and a note that publishing needs its own per-post approval in the publish flow.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: reply key = `run_id`.
10. Postconditions: user informed.
11. Verification: output schema valid.
12. Failure routes: delivery fails → result kept in the run record; `partially_completed`.
13. Escalation triggers: none.
14. Evidence: reply ID.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Ownership | `ownership_confirmed = true` | request | true | ask, then `blocked` |
| Fact fidelity | every fact span verbatim in the brief | S2 | 100 % | drop fact |
| Claim mapping | every factual sentence maps to ≥ 1 fact | S5 claim map | 100 % | redraft once, then `blocked` |
| Specifics | numbers, names, dates, prices present in the brief | S5 | 100 % | redraft |
| No hype | no superlative absent from the brief | S5 | none | redraft |
| Lead with the news | hook contains the `news` subject | S5 | true | redraft |
| Mentions | every third-party name confirmed | S6 records | 100 % | remove and redraft |
| Embargo | not ready-to-publish before `embargo_until` | queue state | true | correct the state |
| Output schema | `announcement_result` valid | validator | valid | fix fields; never fake values |

A well-written draft is not evidence of accuracy; only the claim map against the brief is.

## J. Self-correction and recovery

- **Failure classes:** validation, missing ownership, thin brief, non-verbatim fact, tier unavailable, empty completion, unmapped claim, superlative, buried news, unconfirmed mention, save timeout.
- **Permitted corrections:** ask once for missing facts; drop non-verbatim facts; redraft once with the failures named; remove a declined mention and redraft; reconcile a save by client key.
- **Forbidden corrections:** adding facts the user did not give; web lookups; loosening the claim rules; switching tier silently; queueing a failing draft.
- **Maximum correction cycles:** 1 per step; 2 per run.
- **Reverification required after correction:** yes — S5 re-runs on every new version.
- **Stop conditions:** ownership unconfirmed; claim checks failing after a redraft; budget exhausted.
- **Recovery or compensation:** none needed.
- **What must be reported:** dropped facts, removed mentions, redrafts, anything the brief lacked (for example "no date given, so the post says 'today'" only if the brief said today).

## K. Independent challenge

- **Assumptions to challenge:** "today" or "now" is accurate at publish time (embargo and scheduling may shift it); the user can announce the item (confidentiality, partners).
- **Alternative approach:** for a very thin brief, propose a short "building in public" update instead of a launch-style post.
- **Contradictory evidence:** facts inside the brief that conflict (two prices, two dates) — ask the user rather than choose.
- **Could mislead:** a hook that implies general availability when the brief says beta; rounded or reframed numbers.
- **Separate verifier:** S5 rules are deterministic and independent of the drafter; the optional LLM check uses a different model and can only flag.
- **Findings requiring disclosure:** conflicting facts, beta vs live wording, embargo timing.

## L. Human escalation

- **Triggers:** unconfirmed ownership; third-party mentions; regulated topics (financial results, health or security claims); conflicting facts in the brief; claim checks failing after a redraft.
- **Why the agent cannot decide:** legal, contractual and reputational exposure belongs to the user.
- **Information presented:** the draft, the claim map, the specific conflict or mention.
- **Decision requested:** confirm or remove a mention; choose the correct fact; approve, edit or reject.
- **Options and consequences:** removing a mention weakens the post but avoids an unapproved claim.
- **Safe behaviour while waiting:** nothing queued that has not passed checks.
- **If no response:** after 24 hours the run ends; a queued draft stays `draft`.
- **Resume:** S5 re-runs before any status change.

## M. Completion conditions

Mandatory: `announcement_result` valid; 100 % of claims mapped to the brief; news in the hook; mentions confirmed; nothing published; limitations disclosed; evidence references present.

- `completed` — draft queued (and approved or rejected if the user decided).
- `partially_completed` — draft produced but the queue save or the reply failed, or facts were dropped and disclosed.
- `blocked` — ownership not confirmed, claim checks failing, or unresolved mentions.
- `failed` — extraction or drafting tier unavailable.
- `cancelled` — the user cancelled.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome ("LinkedIn announcement queued — 4 claims, all from your brief"); `announcement_result`; step statuses S1–S9; verification results and failed checks; evidence references (post ID, fact sheet, approval ID and hash, mention confirmations); approvals and scope; side effects (queue rows) and reconciliation status; corrections performed; unresolved challenges; limitations (publishing separate); escalation state; learning candidate. Never include confidential brief text in shared logs.

## O. Learning and revision

- **Outcome metrics:** approval rate, edit distance, redraft rate, claim-check failures by type (unmapped, superlative, buried news), mention confirmations requested.
- **Evidence retained:** queued posts with fact sheets and approval records, per the retention policy.
- **Data minimisation:** metrics without brief or post text.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}` — for example, a phrase the user always removes added to banned phrases (with the user's consent).
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests plus the invented-claim regression set.
- **Promotion approval:** publisher review plus Skill Port review.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (fictional launch brief, LinkedIn) | news-first draft queued; all claims mapped | UNVERIFIED on Skill Port; observed in the reference implementation |
| Happy path (X) | ≤ 270 characters, no link or hashtag | TBD |
| Ownership not confirmed | asks, then `blocked` | TBD |
| Missing or thin brief | asks once what is new | TBD |
| Invalid input (past embargo, bad link) | handled with a note | TBD |
| Missing capability (drafting tier absent) | `failed` with reason; no switch | TBD |
| Permission denied (`content:queue:write` missing) | nothing queued, `blocked` | TBD |
| Unauthorized scope (another tenant's queue) | denied | TBD |
| Tenant isolation | idempotency lookup reads only own posts | TBD |
| Save timeout and replay | reconciled by client key; one draft | TBD |
| False success (draft adds "used by 1,000 teams") | caught by S5; redraft or `blocked` | TBD |
| Superlative not in brief ("the first") | caught by S5 | TBD |
| Buried news (hook is an opinion) | caught by S5 | TBD |
| Third-party mention | confirmation requested; removed if declined | TBD |
| Someone else's news submitted | routed to the commentary recipe | TBD |
| Correction budget exhausted | `blocked` with failures shown | TBD |
| Escalation (regulated topic) | user review recommended before queueing | TBD |
| Cancellation | nothing queued | TBD |
| Rollback / compensation | not applicable — no external side effects; a draft can be rejected | n/a |
| Output schema and status | valid result; correct status | TBD |
| Compatibility / regression (format profile change) | profile version recorded; old drafts re-checked on edit | TBD |

All tests use fictional briefs in a test tenant. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, approval UI and schema registry — TBD (owner: publisher, with Skill Port).
- Detecting "someone else's news" and regulated topics is heuristic; misses are caught only by the user's review (owner: publisher).
- Only LinkedIn and X format profiles exist; other platforms would be new, reviewed variants.
- Publishing and image generation are separate capabilities; X publishing is not covered by this package version.
- Per-draft token cap and queue retention — TBD (owner: operator).
