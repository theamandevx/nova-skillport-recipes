# Publish to X

## A. Identity and ownership

- **Recipe ID:** `nova.content.publish-to-x` (proposed until assigned by Skill Port)
- **Name:** Publish to X
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** marketing
- **Tags:** social-publishing, x, per-post-approval, cost-before-approval, link-pricing, idempotency, human-approval
- **Summary:** Publishes one already-approved post to the user's own X account after a per-post publish approval that shows the exact text, the attached image, the account and the cost (higher when the text contains a link). A claim lock plus reconciliation by post ID make retries safe, so a post is never published twice.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** X API v2 with user-context OAuth 2.0 (authorization code with PKCE) and pay-per-use write access; a content store holding approved posts with a publish state; object storage holding post images; an operator-maintained, date-stamped price table; an append-only usage ledger; an owner notification channel. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — the core flow (approved post, cost shown before approval, publishing with or without an image, and the cost record) runs in Nova's reference implementation; not every control described here has evidence yet (Section P lists the tests still needed). Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.content.posts` | Capability | Read approved posts and track their publish state | medium | UNVERIFIED |
| `nova.content.posts.get` | Operation (read) | Return text, image reference, platform, approval state, content hash, publish state | low | UNVERIFIED |
| `nova.content.posts.claim_publish` | Operation (update, conditional) | `approved` → `publishing` with `claim_id` and lease, only if the content hash matches | medium | UNVERIFIED |
| `nova.content.posts.mark_published` | Operation (update) | `publishing` → `published` with platform post ID, permalink, time | medium | UNVERIFIED |
| `nova.content.posts.release_claim` | Operation (update) | `publishing` → `approved` (confirmed not posted) or `publish_unknown` (unresolved) | medium | UNVERIFIED |
| `nova.content.link_guard` | Capability | Detect links in post text | low | UNVERIFIED |
| `nova.content.link_guard.detect` | Operation (read, deterministic) | URLs, `www.` hosts and bare domains from a curated TLD list; no false positives on version numbers or file names | low | UNVERIFIED |
| `nova.x.pricing` | Capability | Quote the cost of one post | low | UNVERIFIED |
| `nova.x.pricing.quote` | Operation (read) | Price for a post with or without a link, from the operator's date-stamped table | low | UNVERIFIED |
| `nova.x.account` | Capability | The connected X account and its token | medium | UNVERIFIED |
| `nova.x.account.whoami` | Operation (read) | Account ID and handle, granted scopes, token validity | low | UNVERIFIED |
| `nova.x.media` | Capability | Upload images for a post | medium | UNVERIFIED |
| `nova.x.media.upload` | Operation (create) | Upload one image (category `tweet_image`); returns a media ID (not public until attached) | low | UNVERIFIED |
| `nova.x.posts` | Capability | Create and read posts on the user's account | high | UNVERIFIED |
| `nova.x.posts.create` | Operation (create + public external side effect, paid) | Publish one post | high | UNVERIFIED |
| `nova.x.posts.list_recent` | Operation (read) | The account's recent posts, for reconciliation | low | UNVERIFIED |
| `nova.x.posts.get` | Operation (read) | One post by ID, for verification | low | UNVERIFIED |
| `nova.usage.ledger` | Capability | Append-only usage and spend | low | UNVERIFIED |
| `nova.usage.ledger.record` | Operation (create) | One row per published post | low | UNVERIFIED |
| `nova.notify.owner` | Capability | Deliver a message to the requesting user's own channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Ask for approval; report the result | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.x.posts.create.text` — text only. Binding `binding://nova/x/create-post` (UNVERIFIED); protocol: X API v2 create-post operation (`POST /2/tweets`) through a registered adapter; mode: sync.
- `nova.x.posts.create.with-media` — text plus one image; requires a media ID from `nova.x.media.upload` (X API v2 media upload, `POST /2/media/upload`). Same binding.
- **Selection:** deterministic — the post has an image → `with-media`; otherwise `text`. If the image cannot be uploaded, the run stops; it never silently publishes text-only, because that is not the content the user approved.
- `nova.content.link_guard.detect.default`, `nova.x.pricing.quote.default`, and every other operation — only the default variant is supported.

**Authentication (declared requirement):** user-context OAuth 2.0, authorization code with PKCE, scopes exactly `tweet.read`, `tweet.write`, `users.read`, `media.write` (needed only for image posts) and `offline.access` (refresh token). No direct-message or follow scopes. Configure exactly these scopes rather than accepting a credential template's default scope set. Token in the runtime as `secret_ref://x-oauth2-user`.

**Link policy (operator setting `link_policy`):** `confirm` (default) — a post with a link is allowed only after the approval shows the higher link price and the user acknowledges it; `block` — posts with links are refused on X.

**Bindings:** `binding://nova/content/posts`, `binding://nova/content/link-guard`, `binding://nova/x/pricing`, `binding://nova/x/whoami`, `binding://nova/x/media-upload`, `binding://nova/x/create-post`, `binding://nova/x/list-recent`, `binding://nova/x/get-post`, `binding://nova/usage/ledger`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: a header-authenticated publish gateway on the automation hub (download image → upload media → create post) using a generic OAuth 2.0 PKCE credential; link checks run in both the editor and the server; cost recorded per post.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Post | `schema://nova/publish-to-x/post@0.1.0` | the requesting user's tenant | read; update (publish-state fields only) |
| Post image | reference (storage key) | the user's image bucket | read |
| X media | provider media ID | the connected account | create |
| X post | provider post ID | the connected account | create; read |
| Usage row | `schema://nova/publish-to-x/usage-record@0.1.0` | the user's tenant ledger | create |
| Approval | `schema://nova/publish-to-x/approval@0.1.0` | the requesting user | create (by the user) |

**Authorization requirements (declared, never a grant):** `content:posts:read`, `content:posts:publish_state`, `content:assets:read`, `pricing:read`, `x:tweet.write`, `x:tweet.read`, `x:users.read`, `x:media.write` (image posts only), `usage:write`, `notify:owner`.

**Policy / trust:** risk class `high` (public, effectively irreversible, paid, on the user's identity). Requires per-post approval bound to the content hash and quote, a daily post cap per account, the link policy, and compliance with X's developer terms and automation rules (no spam, no duplicate posts, no automated replies or mentions).

## C. Intent and applicability

- **User problem:** An approved X post should go live on the user's account without copy-pasting, with no surprise cost and no risk of a double post.
- **Desired outcome:** The post appears once on the user's X account with the approved text and image; the permalink, cost and evidence are recorded.
- **Trigger or entry conditions:** The user asks to publish a named approved post to X, or presses "Publish to X" on it; or a scheduled post becomes due and the user confirms after the reminder.
- **Applicable situations:** The user's own posts on the user's own connected account.
- **Out-of-scope situations:** Replies, quote posts, threads, mentions, direct messages, follows, likes; posting on behalf of other accounts; bulk or automated posting; publishing a post that is not `approved`; editing or deleting published posts; automatic publishing at a scheduled time without the user.
- **Required inputs:** `post_id` (see Section D).
- **Prerequisites:** A connected X account with the declared scopes and a valid refresh token; pay-per-use write access with credit available; price table loaded and dated; the post approved for platform `x`.
- **Required capabilities:** `nova.content.posts`, `nova.content.link_guard`, `nova.x.pricing`, `nova.x.account`, `nova.x.media`, `nova.x.posts`, `nova.usage.ledger`, `nova.notify.owner` (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port. A credit-balance read for X pay-per-use is not assumed to exist (`TBD`).
- **Assumptions:** The core flow runs in the reference implementation. Assumed: X's current prices and access terms match the operator's table (must be re-checked; prices change).
- **Ambiguity policy:** Stop. Publishing is public and irreversible: if the post, text, image or account is unclear, ask; never guess.
- **Vedaxi task type:** none — this recipe is not one of the seven task types Vedaxi supports today.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `post_id` | string | yes | user / UI | — | Ask once; not found or other tenant → `blocked`. |
| `account_ref` | string | no | user settings | the user's single connected X account | Several accounts and none chosen → ask. |
| `approval` | object `approval@0.1.0` | yes (collected in S5) | the user | — | Missing, expired or hash mismatch → no publish. |

```json
{"$id":"schema://nova/publish-to-x/request@0.1.0","type":"object","required":["post_id"],
 "properties":{"post_id":{"type":"string","minLength":1},"account_ref":{"type":"string"}}}
```

```json
{"$id":"schema://nova/publish-to-x/approval@0.1.0","type":"object",
 "required":["approval_id","post_id","content_hash","account_id","quote","contains_link","approved_at","expires_at"],
 "properties":{"approval_id":{"type":"string"},"post_id":{"type":"string"},"content_hash":{"type":"string"},
 "account_id":{"type":"string"},"contains_link":{"type":"boolean"},"link_acknowledged":{"type":"boolean"},
 "quote":{"type":"object","required":["amount","currency","price_table_as_of"],"properties":{"amount":{"type":"number","minimum":0},"currency":{"type":"string"},"price_table_as_of":{"type":"string","format":"date"}}},
 "approved_at":{"type":"string","format":"date-time"},"expires_at":{"type":"string","format":"date-time"}}}
```

`content_hash` = sha256 of the normalised text plus the image's content hash (or `none`). Any edit to the text or image changes it and invalidates the approval.

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `publish_result` | object `schema://nova/publish-to-x/publish-result@0.1.0` | `post_id`, `status`, `publish_state` | the post record, the user's channel and the run record | public once published | read-back of the live post (Section I) |

```json
{"$id":"schema://nova/publish-to-x/publish-result@0.1.0","type":"object","required":["post_id","status","publish_state"],
 "properties":{"post_id":{"type":"string"},"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},
 "publish_state":{"enum":["approved","publishing","published","publish_unknown"]},"variant":{"enum":["text","with-media"]},
 "platform_post_id":{"type":["string","null"]},"permalink":{"type":["string","null"],"format":"uri"},
 "published_at":{"type":["string","null"],"format":"date-time"},"account_handle":{"type":["string","null"]},
 "contains_link":{"type":"boolean"},"approval_id":{"type":["string","null"]},"reconciled":{"type":"boolean"},
 "cost":{"type":["object","null"],"required":["amount","currency","basis"],"properties":{"amount":{"type":"number","minimum":0},"currency":{"type":"string"},"basis":{"enum":["price_table","provider_reported"]}}},
 "usage_record_id":{"type":["string","null"]}}}
```

Supporting schemas:

```json
{"$id":"schema://nova/publish-to-x/post@0.1.0","type":"object","required":["post_id","tenant_id","platform","status","text","content_hash","publish_state"],"properties":{"post_id":{"type":"string"},"tenant_id":{"type":"string"},"platform":{"const":"x"},"status":{"enum":["draft","approved","rejected"]},"text":{"type":"string","minLength":1},"image_ref":{"type":["string","null"]},"content_hash":{"type":"string"},"publish_state":{"enum":["approved","publishing","published","publish_unknown"]},"claim_id":{"type":["string","null"]},"claimed_at":{"type":["string","null"],"format":"date-time"},"lease_expires_at":{"type":["string","null"],"format":"date-time"},"platform_post_id":{"type":["string","null"]},"permalink":{"type":["string","null"]},"published_at":{"type":["string","null"],"format":"date-time"},"last_publish_error":{"type":["string","null"]},"scheduled_for":{"type":["string","null"],"format":"date-time"}}}
{"$id":"schema://nova/publish-to-x/link-check@0.1.0","type":"object","required":["contains_link","matches"],"properties":{"contains_link":{"type":"boolean"},"matches":{"type":"array","items":{"type":"object","required":["text","kind"],"properties":{"text":{"type":"string"},"kind":{"enum":["url","www","bare_domain"]}}}}}}
{"$id":"schema://nova/publish-to-x/quote@0.1.0","type":"object","required":["amount","currency","contains_link","price_table_as_of"],"properties":{"amount":{"type":"number","minimum":0},"currency":{"type":"string"},"contains_link":{"type":"boolean"},"price_table_as_of":{"type":"string","format":"date"}}}
{"$id":"schema://nova/publish-to-x/account@0.1.0","type":"object","required":["account_id","handle","scopes","token_valid"],"properties":{"account_id":{"type":"string"},"handle":{"type":"string"},"scopes":{"type":"array","items":{"type":"string"}},"token_valid":{"type":"boolean"},"checked_at":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/publish-to-x/media@0.1.0","type":"object","required":["media_id","image_hash"],"properties":{"media_id":{"type":"string"},"image_hash":{"type":"string"},"media_category":{"const":"tweet_image"},"uploaded_at":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/publish-to-x/usage-record@0.1.0","type":"object","required":["usage_id","ts","provider","cost_usd","cost_basis","meta","idempotency_key"],"properties":{"usage_id":{"type":"string"},"ts":{"type":"string","format":"date-time"},"provider":{"type":"string"},"model":{"type":["string","null"]},"units":{"type":"object","properties":{"posts":{"const":1},"with_link":{"type":"boolean"},"with_media":{"type":"boolean"}}},"cost_usd":{"type":"number"},"cost_basis":{"enum":["provider_reported","estimate","price_table"]},"meta":{"type":"object","required":["job"],"properties":{"job":{"const":"content:publish-to-x"},"run_id":{"type":"string"},"adjusts_usage_id":{"type":["string","null"]}},"additionalProperties":{"type":["string","null"]}},"idempotency_key":{"type":"string"}}}
{"$id":"schema://nova/publish-to-x/delivery@0.1.0","type":"object","required":["message_id","status"],"properties":{"message_id":{"type":"string"},"channel":{"type":"string"},"status":{"enum":["delivered","queued","failed"]},"delivered_at":{"type":["string","null"],"format":"date-time"}}}
```

**Example (fictional)**

Input: `{"post_id":"post_0051"}` — text "Shipped: search results now load in under 200 ms. Small change, big difference.", one image, no link.

Output: `{"post_id":"post_0051","status":"completed","publish_state":"published","variant":"with-media","platform_post_id":"1000000000000000001","permalink":"https://example.com/x/status/1000000000000000001","published_at":"2026-05-04T09:12:31Z","account_handle":"example_brand","contains_link":false,"approval_id":"apr_0090","reconciled":false,"cost":{"amount":0.02,"currency":"USD","basis":"price_table"},"usage_record_id":"use_1204"}` (the permalink stands in for the platform's real permalink; the price is illustrative, not X's current price).

## E. Constraints and budgets

- **Hard constraints:** only posts with status `approved` and platform `x`; per-post publish approval bound to the content hash, account and quote, unexpired; the quote shown before approval, with the higher link price when a link is present; link policy enforced; weighted text length ≤ 280 under X's counting rules; the user's own connected account only; at most one publish per post (claim lock); never publish text-only when the approved post has an image; never retry `create` on an ambiguous outcome before reconciliation; daily post cap per account (`TBD`); no replies, mentions, threads or messages.
- **Soft constraints:** publish at the user's chosen time; prefer posts without links on X.
- **Resource and tenant scope:** the requesting user's posts, images, connected X account and ledger.
- **Data sources allowed:** the post record, its image, the price table, the account's own recent posts (reconciliation).
- **External transfers permitted:** the approved text and image to X only.
- **Time limit:** create call ≤ 20 s; reconciliation ≤ 90 s; whole run ≤ 45 minutes including approval.
- **Cost or token budget:** one post per approval at the quoted price; daily spend cap `TBD` (operator).
- **Maximum steps:** 11 (S1–S11).
- **Maximum retries and correction cycles:** `create`: 1 attempt, never retried automatically; media upload: 2 retries (unattached media is never public); reads: 2 retries; reconciliation: 3 checks.
- **Rate limits and concurrency:** one publish in flight per account; X API rate limits apply (`TBD` per access tier).
- **Retention and logging constraints:** keep approval, claim and result records with the post; logs never contain OAuth tokens.
- **Prohibited actions:** editing post text at publish time; deleting posts; posting to an account other than the approved one; automatic publishing of scheduled posts.
- **Stop conditions:** post not approved or already published; approval missing, expired or mismatched; token invalid; insufficient pay-per-use credit; link blocked by policy; outcome unresolved after reconciliation.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | X media (one per image) | approval valid; claim held | Approval Required (covered by S5) | media ID returned |
| Create | X post (one per post) | claim held; approval valid | Approval Required (S5) | read-back by post ID |
| Create | Usage row | post confirmed published | Allowed | row readable |
| Read | Post, image, price table, account, recent posts | own tenant and own account | Allowed | — |
| Update | Post: `publish_state`, `claim_id`, `claimed_at`, `lease_expires_at`, `platform_post_id`, `permalink`, `published_at`, `last_publish_error` | conditional on state and content hash | Allowed | read-back |
| Delete | any (local or on X) | — | Not Allowed | — |
| External side effect | public post on X; per-post charge | all hard constraints pass | Approval Required (S5) | post readable and matches |
| External side effect | approval request and result message to the user | — | Allowed | delivery receipt |

- **Idempotency:** one publish per (`post_id`, platform). `claim_publish` is a compare-and-set from `approved` with a matching `content_hash`; a second request fails the claim and receives the current state. The usage row key is (`post_id`, `platform_post_id`). The platform's own duplicate-content rejection is not relied upon.
- **Expected side effects:** a public post on the user's timeline; a charge per post (higher with a link).
- **Compensation:** none automatic. If the user wants a published post removed, they delete it on X or through a separately approved action outside this recipe; the charge is not refunded (`TBD` per X terms).
- **Irreversible:** the post becoming public (followers may see or copy it immediately) and its charge.
- **Ambiguous outcomes:** on a timeout, 5xx or dropped connection from `create`, S9 reads the account's recent posts since `claimed_at` and matches the normalised text hash (and media presence); found → adopt that post ID; not found after three checks over about 60 s → `publish_unknown`, no retry, and the user is asked to check. A claim whose lease expired is reconciled the same way before it is ever released.

## G. Autonomy and approval policy

- **Default autonomy level:** A1 — supervised execution.
- **Planning level:** A2 (eligibility, link check, quote, readiness).
- **Execution level:** A1 — publishing only after the S5 approval, one post per approval.
- **Verification level:** A2.
- **Recovery level:** A1 — no retry of `create` without reconciliation and, after an unknown outcome, a new approval.
- **Approval required for:** every publish (text, image, account, quote, link acknowledgement).
- **Actions never permitted:** publishing without approval or with a stale approval; publishing an edited post on an old approval; posting text-only instead of an approved image post; replies, mentions, DMs, follows; deleting posts; auto-publishing at a scheduled time.
- **Conditions requiring clarification:** several connected accounts; post platform not `x`.
- **Conditions requiring escalation:** token refresh fails (reconnect needed); insufficient credit; `publish_unknown`; daily cap reached.
- **Safe state after stop, cancellation or escalation:** nothing published (or the single post published and recorded), no claim left without reconciliation, the user told the exact state.

Approval covers one post with the stated text, image, account and price. It does not cover retries after an unknown outcome or later edits.

## H. Execution plan

**S1 — Validate the request**
1. Objective: a schema-valid request.
2. Capability: none (local validation).
3. Input mapping: user action → `request@0.1.0`.
4. Dependencies: none.
5. Preconditions: explicit publish request.
6. Action / side effects: validate; no side effects.
7. Autonomy: A2; no approval.
8. Timeout / retry: 5 s / none.
9. Idempotency: not applicable.
10. Postconditions: valid request.
11. Verification: JSON Schema passes.
12. Failure routes: missing `post_id` → ask once → else `blocked`.
13. Escalation triggers: none.
14. Evidence: validated request.

**S2 — Load the post and check eligibility**
1. Objective: confirm this post may be published.
2. Capability: `nova.content.posts.get`.
3. Input mapping: `post_id`.
4. Dependencies: S1.
5. Preconditions: request valid.
6. Action: read; compute `content_hash`; check status `approved`, platform `x`, `publish_state = approved`, weighted length ≤ 280, image type and size within X's limits (`TBD`).
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: eligible post with hash.
11. Verification: every check recorded.
12. Failure routes: already `published` → return the existing permalink (`completed`, no action); `publishing` or `publish_unknown` → `blocked` with the state; not approved → `blocked`.
13. Escalation: none.
14. Evidence: post state, `content_hash`.

**S3 — Link check and cost quote**
1. Objective: know whether the text has a link and what the post will cost.
2. Capability: `nova.content.link_guard.detect`; `nova.x.pricing.quote`.
3. Input mapping: post text.
4. Dependencies: S2.
5. Preconditions: eligible post.
6. Action: detect URLs, `www.` hosts and bare domains from the curated TLD list (version numbers such as "3.5" and file names such as "Node.js" do not match); quote the matching price with its `as_of` date.
7. Autonomy: A2.
8. Timeout / retry: 5 s / 1 retry.
9. Idempotency: read-only.
10. Postconditions: `link-check`, `quote`.
11. Verification: price table `as_of` within the operator's freshness window (`TBD`, default 30 days).
12. Failure routes: link and policy `block` → `blocked` ("remove the link or post it elsewhere"); stale price table → `blocked` until refreshed.
13. Escalation: stale prices → tell the operator.
14. Evidence: matches, quote, table date.

**S4 — Account readiness**
1. Objective: the right account is connected with the right scopes.
2. Capability: `nova.x.account.whoami`.
3. Input mapping: `account_ref`.
4. Dependencies: S3.
5. Preconditions: quote ready.
6. Action: refresh the token if needed; read account ID, handle and granted scopes.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 1 retry.
9. Idempotency: read-only.
10. Postconditions: account identity; scopes include `tweet.write` (and `media.write` for an image).
11. Verification: account = the user's configured account.
12. Failure routes: refresh fails → `blocked`, "reconnect your X account"; missing scope → `blocked`.
13. Escalation: reconnect requires the user's browser sign-in.
14. Evidence: account ID, scope list (no tokens).

**S5 — Publish approval**
1. Objective: explicit approval for this exact post at this price.
2. Capability: `nova.notify.owner.send` (approval prompt).
3. Input mapping: text, image preview, account handle, quote, link note.
4. Dependencies: S4.
5. Preconditions: all checks passed.
6. Action: ask "Publish to example_brand on X? Text and image below. Cost about 0.02 USD." — with a link: "Contains a link: cost about 0.25 USD instead of 0.02. Publish anyway?" (Approve / Cancel).
7. Autonomy: A0 — human decision.
8. Timeout / retry: approval valid 30 minutes; no answer → `cancelled`.
9. Idempotency: the approval ID binds `content_hash`, account and quote.
10. Postconditions: `approval@0.1.0`.
11. Verification: approval hash = current `content_hash`; link acknowledged when a link is present.
12. Failure routes: Cancel or timeout → `cancelled`.
13. Escalation: not applicable.
14. Evidence: approval record.

**S6 — Claim the publish lock**
1. Objective: guarantee a single publisher for this post.
2. Capability: `nova.content.posts.claim_publish`.
3. Input mapping: `post_id`, `content_hash`, new `claim_id`, lease 10 minutes.
4. Dependencies: S5.
5. Preconditions: approval valid.
6. Action / side effects: compare-and-set `approved` → `publishing`.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries (a retry re-reads the state first).
9. Idempotency: compare-and-set; only one claim can win.
10. Postconditions: claim held by this run.
11. Verification: read-back shows this `claim_id`.
12. Failure routes: claim lost → stop and report the current state; hash mismatch (post edited) → `blocked`, approval void.
13. Escalation: none.
14. Evidence: `claim_id`, `claimed_at`.

**S7 — Upload media (image posts only)**
1. Objective: a media ID for the approved image.
2. Capability: `nova.x.media.upload`.
3. Input mapping: image bytes from storage (hash must match the approved image).
4. Dependencies: S6.
5. Preconditions: claim held; `media.write` granted.
6. Action / side effects: upload; unattached media is not public.
7. Autonomy: A1 (covered by S5).
8. Timeout / retry: 60 s / 2 retries.
9. Idempotency: safe to repeat — only the media ID used in S8 matters.
10. Postconditions: media ID.
11. Verification: media ID present; image hash = approved hash.
12. Failure routes: exhausted → `release_claim` to `approved` (nothing posted), `failed`; never fall back to text-only.
13. Escalation: repeated 403 → check `media.write` scope with the user.
14. Evidence: media ID.

**S8 — Create the post**
1. Objective: publish exactly once.
2. Capability: `nova.x.posts.create.text` or `.with-media`.
3. Input mapping: approved text; media ID if any.
4. Dependencies: S6 (and S7).
5. Preconditions: claim held; approval unexpired; daily cap not reached.
6. Action / side effects: the post goes public; the charge accrues.
7. Autonomy: A1 (covered by S5).
8. Timeout / retry: 20 s / **no automatic retry**.
9. Idempotency: claim lock; reconciliation on any unclear result.
10. Postconditions: platform post ID, or a clear rejection, or an ambiguous outcome.
11. Verification: response contains a post ID.
12. Failure routes: clear rejection (4xx with reason, including insufficient credit) → `release_claim` to `approved`, `failed` or `blocked` with the reason; ambiguous → S9.
13. Escalation: insufficient credit → tell the user.
14. Evidence: response status, post ID or error.

**S9 — Reconcile an ambiguous outcome (only when S8 is unclear or a lease expired)**
1. Objective: learn whether the post went live without posting again.
2. Capability: `nova.x.posts.list_recent`.
3. Input mapping: account ID, `claimed_at`, normalised text hash, media presence.
4. Dependencies: S8.
5. Preconditions: claim held.
6. Action: wait 15 s; read up to 20 recent posts since `claimed_at`; match; repeat up to 3 times.
7. Autonomy: A2.
8. Timeout / retry: 90 s total.
9. Idempotency: read-only.
10. Postconditions: matched post ID, or unresolved.
11. Verification: text hash and media presence both match.
12. Failure routes: unresolved → `release_claim` to `publish_unknown`; run `failed`; no retry.
13. Escalation: ask the user to check the timeline; a retry later needs a new approval and a fresh reconciliation.
14. Evidence: checks performed, match or none.

**S10 — Mark published and record cost**
1. Objective: record the live post and its charge once.
2. Capability: `nova.content.posts.mark_published`; `nova.usage.ledger.record`.
3. Input mapping: post ID, permalink, time; quote; `meta.job = content:publish-to-x`.
4. Dependencies: S8 or S9 with a post ID.
5. Preconditions: post ID known.
6. Action / side effects: `publishing` → `published`; append one usage row.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 3 retries.
9. Idempotency: conditional on `claim_id`; usage key (`post_id`, `platform_post_id`).
10. Postconditions: record `published`; one usage row.
11. Verification: read-back of both.
12. Failure routes: write fails after retries → `partially_completed` (live but not recorded); keep the post ID in the run record and retry on the next run.
13. Escalation: none.
14. Evidence: post version, `usage_id`.

**S11 — Verify and deliver**
1. Objective: prove the live post matches the approval and tell the user.
2. Capability: `nova.x.posts.get`; `nova.notify.owner.send`.
3. Input mapping: platform post ID.
4. Dependencies: S10.
5. Preconditions: published.
6. Action: read the post; compare text, author and media count; send permalink, cost and any reconciliation note.
7. Autonomy: A2.
8. Timeout / retry: 10 s, 2 retries; delivery 15 s, 2 retries.
9. Idempotency: message key = `post_id` + `published`.
10. Postconditions: verified and delivered.
11. Verification: text equal (after platform URL shortening), author = account, media count = 1 for image posts.
12. Failure routes: mismatch → report it plainly (`partially_completed`); never edit or delete automatically.
13. Escalation: mismatch → ask the user.
14. Evidence: read-back snapshot, message ID.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Input validity | request matches `request@0.1.0` | validator | valid | ask once, then `blocked` |
| Eligibility | status `approved`, platform `x`, state `approved`, length ≤ 280 | S2 | all pass | `blocked` |
| Cost shown | approval contains the quote; link acknowledged when present | approval record | present | no publish |
| Approval scope | approval hash = current content hash; unexpired | S5 vs S6 | equal | `blocked` |
| Single publisher | exactly one successful claim | claim record | one | stop |
| Content integrity | published text and image = approved | S11 read-back | equal | report; escalate |
| No double post | at most one live post per `post_id` | recent posts, records | ≤ 1 | escalate |
| Cost recorded | one usage row per published post | ledger | 1:1 | retry; report |
| Account | author = configured account | S11 | equal | escalate |
| Output schema | `publish_result` valid | validator | valid | fix fields; never fake values |

A 2xx from `create` proves only that X accepted the request; S11's read-back proves what is live.

## J. Self-correction and recovery

- **Failure classes:** validation, ineligible post, link blocked, stale price table, token or scope failure, insufficient credit, claim conflict, media upload failure, clear rejection, ambiguous outcome, record-write failure, content mismatch.
- **Permitted corrections:** refresh the token; retry reads and media uploads (bounded); reconcile ambiguous outcomes; retry record writes.
- **Forbidden corrections:** retrying `create` before reconciliation; posting text-only instead of an image post; trimming or rewriting text to fit 280; removing a link silently; reusing an expired or mismatched approval; deleting a post.
- **Maximum correction cycles:** 1 per step; 3 reconciliation checks.
- **Reverification required after correction:** yes — S2 eligibility and the approval hash before any later attempt.
- **Stop conditions:** `publish_unknown`; hard constraint fails.
- **Recovery or compensation:** release the claim to `approved` only when the post is confirmed not live.
- **What must be reported:** whether the post is live, its permalink, the cost, any reconciliation, and any unresolved state.

## K. Independent challenge

- **Assumptions to challenge:** the price table is current; the link detector caught every link; the account is the one the user meant.
- **Alternative approach:** for a link-heavy post, would posting without the link (link in a reply by the user) or on another platform be better? Mention it at S5 when a link is present.
- **Contradictory evidence:** a matching post already on the timeline before S8 (posted manually by the user) — then stop and ask.
- **Could mislead:** "published" when only `create` returned; a cost shown from an outdated table.
- **Separate verifier:** the S11 read-back of the live post through the read API, independent of the `create` response; a second link pass with a general URL parser on the final text.
- **Findings requiring disclosure:** reconciliation used, price table age, platform URL shortening differences.

## L. Human escalation

- **Triggers:** every publish (approval); several accounts; token refresh failure; insufficient credit; link blocked; daily cap reached; `publish_unknown`; content mismatch after publishing.
- **Why the agent cannot decide:** public, irreversible, paid actions on the user's identity.
- **Information presented:** text, image preview, account handle, quote with table date, link matches, current publish state, reconciliation evidence.
- **Decision requested:** approve or cancel; reconnect the account; top up credit; confirm whether an unknown post is live.
- **Options and consequences:** for example, a link raises the cost; retrying an unknown outcome risks a double post unless the timeline is checked first.
- **Safe behaviour while waiting:** nothing published; claims held only while reconciling.
- **If no response:** approval expires after 30 minutes → `cancelled`; a `publish_unknown` post stays blocked until the user confirms.
- **Resume:** a new approval, then S2–S4 re-run before S6.

## M. Completion conditions

Mandatory: `publish_result` valid; publish approved with the cost shown; at most one live post per `post_id`; live content verified against the approval; usage row recorded; limitations disclosed; evidence references present.

- `completed` — the post is live, verified, recorded with its cost, and the user was notified (or it was already published and the existing permalink returned).
- `partially_completed` — the post is live but recording, verification or delivery failed, or the read-back showed a difference.
- `blocked` — ineligible post, link policy, stale prices, token or scope failure, insufficient credit, daily cap, or a claim held elsewhere.
- `failed` — X rejected the post, media upload failed, or the outcome stayed unknown after reconciliation (`publish_unknown`).
- `cancelled` — the user declined or approval expired.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome; `publish_result`; step statuses S1–S11; verification results and failed checks; evidence references (approval ID, `claim_id`, platform post ID, `usage_id`, reconciliation checks); approvals and scope; side effects (public post, charge) and reconciliation status; corrections; unresolved challenges; limitations (price table date); escalation state; learning candidate. Never include OAuth tokens or refresh tokens.

## O. Learning and revision

- **Outcome metrics:** approval-to-publish rate, publish success rate, reconciliation rate, `publish_unknown` count, link-post share and cost, cost per post.
- **Evidence retained:** approval, claim and result records; usage rows.
- **Data minimisation:** metrics without post text; recent-post reads used only for matching and not stored.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}` (for example a TLD list addition after a missed link).
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review; approval and idempotency rules cannot be relaxed automatically.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (test account, text and image posts) | one live post each, verified, cost recorded | UNVERIFIED on Skill Port; observed in the reference implementation |
| Missing input (`post_id` absent) | asks once, then `blocked` | TBD |
| Invalid input (post not approved; text over 280 weighted) | `blocked` at S2 | TBD |
| Link detection (URL, `www.`, bare domain; "3.5", "Node.js" negatives) | links found; no false positives; higher quote shown | TBD |
| Link policy `block` | `blocked` before approval | TBD |
| Missing capability (create-post binding absent) | `blocked` with a dependency message | TBD |
| Permission denied (`tweet.write` or `media.write` missing) | `blocked` at S4 | TBD |
| Unauthorized scope (another tenant's post or account) | denied | TBD |
| Tenant isolation | cannot claim or read another tenant's posts | TBD |
| Stale approval (post edited after approval) | hash mismatch → `blocked` | TBD |
| Timeout on `create` | reconciliation; no second post | TBD |
| Duplicate / replay (two publish requests at once; retry after crash) | one claim wins; one live post | TBD |
| False success (2xx but read-back differs or missing) | not reported `completed` | TBD |
| Failed verification (author or media mismatch) | `partially_completed`, escalated | TBD |
| Media upload failure | claim released; no text-only post | TBD |
| Correction budget exhausted (reconciliation unresolved) | `publish_unknown`, `failed`, no retry | TBD |
| Escalation (token refresh fails) | "reconnect" request; nothing published | TBD |
| Cancellation (user declines at S5) | `cancelled`; nothing published | TBD |
| Compensation (claim released only after confirming not live) | state `approved`, no post | TBD |
| Output schema and status | valid `publish_result`, correct status | TBD |
| Compatibility / regression (X API or price change) | endpoints and price table re-validated | TBD |

All tests use a dedicated test X account and sandbox tenant; never a real customer's account. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, schema registry and approval UI — TBD (owner: publisher, with Skill Port).
- X pay-per-use prices, access tier, rate limits and whether a credit-balance read exists — TBD (owner: operator; prices change and the table must be dated).
- Daily post cap and price-table freshness window — TBD (owner: operator).
- OAuth 1.0a user context also works on X API v2 endpoints; it is not declared as a variant in this draft.
- Weighted character counting must follow X's official text rules; the library is an unresolved dependency.
- Image size and format limits — TBD from X's current documentation.
- Threads, replies and video are out of scope.
