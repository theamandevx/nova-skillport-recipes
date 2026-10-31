# Post image

## A. Identity and ownership

- **Recipe ID:** `nova.content.post-image` (proposed until assigned by Skill Port)
- **Name:** Post image
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** design
- **Tags:** image-generation, social-post, layered-composition, platform-sizes, safe-zone, cost-tracking, provider-variants
- **Summary:** For one existing post draft, the agent writes an image prompt that follows a five-layer composition rule and the target platform's size and safe zone, generates one image with an explicitly chosen provider variant (cost shown first), stores the file, attaches it to the post and records the cost of that image.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** A content store holding post drafts; two image-generation providers exposed as separate variants; an LLM for prompt composition; durable object storage; an append-only usage ledger; an owner notification channel. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — the core flow (layered prompt, generation with a chosen provider, storage, attachment and cost record) runs in Nova's reference implementation; not every control described here has evidence yet (Section P lists the tests still needed). Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.content.posts` | Capability | Read a post draft and attach an image to it | low | UNVERIFIED |
| `nova.content.posts.get` | Operation (read) | Return one post: platform, stage, hook, body, current image fields | low | UNVERIFIED |
| `nova.content.posts.attach_image` | Operation (update) | Set `image_ref`, `image_prompt`, `image_status`, `image_meta`; the previous image stays in history | low | UNVERIFIED |
| `nova.content.image_prompt` | Capability | Turn a post into a layered, platform-sized image prompt | low | UNVERIFIED |
| `nova.content.image_prompt.compose` | Operation (read, LLM) | Produce headline, rows, motif, badge, brand layer and size/safe-zone instructions | low | UNVERIFIED |
| `nova.content.image` | Capability | Generate one image with an explicitly chosen provider | medium | UNVERIFIED |
| `nova.content.image.list_providers` | Operation (read) | Return each provider's `ready`, `reason`, supported sizes and price per image | low | UNVERIFIED |
| `nova.content.image.generate` | Operation (create + paid external call) | Generate one image from a verified prompt | medium | UNVERIFIED |
| `nova.content.assets` | Capability | Store binary assets durably | low | UNVERIFIED |
| `nova.content.assets.upload` | Operation (create) | Write the image to the image bucket under a content-hash path | low | UNVERIFIED |
| `nova.usage.ledger` | Capability | Append-only record of AI usage and spend | low | UNVERIFIED |
| `nova.usage.ledger.record` | Operation (create) | Append one usage row per generated image | low | UNVERIFIED |
| `nova.notify.owner` | Capability | Deliver a message to the requesting user's own channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Send the review package to the user | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.content.image.generate.provider-a` — a multimodal model with image output, called through a standard chat-completions interface that returns the image as a base64 data URL. Native portrait output is close to 4:5 (about 0.81). Better at dense, legible text cards. Binding `binding://nova/content/image-provider-a` (UNVERIFIED); protocol: provider HTTPS API through a registered adapter; mode: sync.
- `nova.content.image.generate.provider-b` — a dedicated image-generation API with fixed sizes only: 1024×1024 (1:1), 1024×1536 (2:3) and 1536×1024 (3:2), plus a `quality` setting (`low`, `medium`, `high`). Strong photoreal and product-hero renders; cost depends on size and quality. Binding `binding://nova/content/image-provider-b` (UNVERIFIED); protocol: provider HTTPS API through a registered adapter; mode: sync.
- **Selection policy:** the `provider` input if given, else the operator's default for the preset. Eligibility filters, in order: provider `ready: true`; preset supported directly or through the declared crop mapping (table below); estimate ≤ per-image cap. **No fallback.** The two variants differ in cost, aspect handling and output quality, so if the chosen one is not ready, refuses or fails, the run stops `blocked` and offers the other variant with its own estimate and crop note. Switching needs a new, explicit choice.
- `nova.content.image_prompt.compose.default`, `nova.content.posts.get.default`, `nova.content.posts.attach_image.default`, `nova.content.assets.upload.default`, `nova.usage.ledger.record.default`, `nova.notify.owner.send.default` — only the default variant is supported.

**Five-layer composition rule (applied by `compose`, checked in S5):**

1. **L1 Atmosphere** — background colour, one accent radial glow, a subtle dot-grid or grain texture at 5–8 % opacity and a soft vignette. Never a flat slab.
2. **L2 Hero motif** — one abstract graphic that visually argues the headline (for example crossing arrows for "faster and cheaper"). No axis labels, small numbers or fine print, because generators garble them.
3. **L3 Headline** — the exact headline text, dominant and legible, within the preset's word cap.
4. **L4 Supporting rows** — short rows, each with a simple line icon (never plain arrows or bullets). Facts only from the post.
5. **L5 Brand** — the user's own logomark and wordmark from the brand kit, plus a topical pill badge naming the subject in text. **Never draw real third-party logos** (they garble and raise trademark issues); the named pill and that subject's accent colour are the reference.

**Platform presets:**

| Preset | Target | Provider A request | Provider B size | Safe zone | Density |
|---|---|---|---|---|---|
| `linkedin-portrait` | 4:5 feed image (1080×1350) | portrait, about 4:5 | 1024×1536 (2:3, taller than 4:5) | all text and logos inside the central 4:5 band: top and bottom about 8.3 % clear on 2:3 output, 4 % on provider A output | badge → headline ≤ 12 words → subhead → 3–4 icon rows → motif → logomark |
| `x-landscape` | 16:9 timeline card (1200×675) | landscape | 1536×1024 (3:2) | central 16:9 band; top and bottom 12 % empty | headline ≤ 8 words, ≤ 3 icon chips, bolder motif, dark background |
| `square` | 1:1 generic | square | 1024×1024 | 6 % margin on every side | headline ≤ 10 words, ≤ 3 rows |

**Bindings:** `binding://nova/content/posts`, `binding://nova/content/image-prompt`, `binding://nova/content/image-providers`, `binding://nova/content/image-provider-a`, `binding://nova/content/image-provider-b`, `binding://nova/content/assets`, `binding://nova/usage/ledger`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: the post store sits behind a database dispatcher function; provider A is called directly from the content app and provider B through a webhook gateway on the automation hub; images go to a dedicated storage bucket.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Post | `schema://nova/post-image/post@0.1.0` | the requesting user's tenant | read; update (image fields only) |
| Image prompt | `schema://nova/post-image/image-prompt@0.1.0` | linked to one post | create |
| Image asset | `schema://nova/post-image/image-result@0.1.0` (reference) | the user's image bucket | create |
| Usage row | `schema://nova/post-image/usage-record@0.1.0` | the user's tenant ledger | create |
| Brand kit | `schema://nova/post-image/brand-kit@0.1.0` | the requesting user | read |

**Authorization requirements (declared, never a grant):** `content:posts:read`, `content:posts:write_image` (image fields only), `content:assets:write`, `image:generate:provider-a` or `image:generate:provider-b` (only the chosen one), `usage:write`, `notify:owner`. Provider keys live in the runtime as `secret_ref://image-provider-a` and `secret_ref://image-provider-b`; storage as `secret_ref://object-storage`.

**Policy / trust:** risk class `medium` (paid generation; brand and trademark exposure once the post is published). Requires a per-image cost cap, a daily image budget and at most 3 generations per post per run. Publishing is out of scope.

## C. Intent and applicability

- **User problem:** A post needs an image that carries its message, fits the platform's crop and looks on-brand, without the user designing it.
- **Desired outcome:** One verified image attached to the post, with the prompt, provider, size and cost recorded.
- **Trigger or entry conditions:** The user asks for an image for a named post, or presses "generate image" on a post draft.
- **Applicable situations:** Content cards, announcement covers, topic cards for LinkedIn or X, generic square images.
- **Out-of-scope situations:** Photoreal images of real identifiable people; reproducing third-party logos, mascots or copyrighted characters; editing user-supplied photos; video; publishing the post; bulk generation across many posts.
- **Required inputs:** `post_id` (see Section D).
- **Prerequisites:** A post with text in the user's tenant; at least one ready image provider; a brand kit (or acceptance that L5 shows only the badge); storage and ledger available.
- **Required capabilities:** `nova.content.posts`, `nova.content.image_prompt`, `nova.content.image`, `nova.content.assets`, `nova.usage.ledger`, `nova.notify.owner` (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port; every binding is a dependency to resolve before activation. The vision check of the rendered text (Section K) needs a vision-capable model binding.
- **Assumptions:** The core flow runs in the reference implementation. Assumed: the target runtime offers equivalent providers with the same size limits.
- **Ambiguity policy:** Use a documented safe default for preset (from the post's platform) and provider (operator default); ask when the post has no platform or no text; never invent headline facts.
- **Vedaxi task type:** none — this recipe is not one of the seven task types Vedaxi supports today.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `post_id` | string | yes | user / UI; internal | — | Ask once; unknown or other tenant's post → `blocked`. |
| `platform_preset` | enum `linkedin-portrait`, `x-landscape`, `square` | no | user | from the post's platform | Contradicts the post's platform → ask which to use. |
| `provider` | enum `provider-a`, `provider-b` | no | user | operator default for the preset | Not ready → `blocked` with reason; never switched silently. |
| `quality` | enum `low`, `medium`, `high` | no | user | operator default | Ignored for provider A (disclosed). |
| `brand_kit_ref` | string | no | user settings | the user's default kit | Missing kit → L5 shows only the badge; disclosed. |
| `topic_badge` | string, 1–24 chars | no | user | derived from the post subject | Longer → truncated at a word boundary and shown. |
| `style_notes` | string, ≤ 300 chars | no | user | — | Requests a real logo or a real person's likeness → refused, rest kept. |
| `max_cost_usd` | number > 0 | no | user | operator per-image cap | Above the operator cap → clamped to the cap. |

```json
{"$id":"schema://nova/post-image/request@0.1.0","type":"object","required":["post_id"],
 "properties":{"post_id":{"type":"string","minLength":1},
 "platform_preset":{"enum":["linkedin-portrait","x-landscape","square"]},
 "provider":{"enum":["provider-a","provider-b"]},"quality":{"enum":["low","medium","high"]},
 "brand_kit_ref":{"type":"string"},"topic_badge":{"type":"string","minLength":1,"maxLength":24},
 "style_notes":{"type":"string","maxLength":300},"max_cost_usd":{"type":"number","exclusiveMinimum":0}}}
```

```json
{"$id":"schema://nova/post-image/image-prompt@0.1.0","type":"object","required":["preset","size","headline","layers","safe_zone","prompt_text"],
 "properties":{"preset":{"enum":["linkedin-portrait","x-landscape","square"]},"size":{"type":"object","required":["w","h"],"properties":{"w":{"type":"integer"},"h":{"type":"integer"}}},
 "headline":{"type":"string","minLength":1,"maxLength":90},"layers":{"type":"object","required":["atmosphere","motif","headline","rows","brand"],
  "properties":{"atmosphere":{"type":"string"},"motif":{"type":"string"},"headline":{"type":"string"},
  "rows":{"type":"array","maxItems":4,"items":{"type":"object","required":["icon","text"],"properties":{"icon":{"type":"string"},"text":{"type":"string","maxLength":60}}}},
  "brand":{"type":"object","required":["badge"],"properties":{"badge":{"type":"string"},"wordmark":{"type":["string","null"]}}}}},
 "safe_zone":{"type":"object","properties":{"top_pct":{"type":"number"},"bottom_pct":{"type":"number"},"side_pct":{"type":"number"}}},"prompt_text":{"type":"string","maxLength":4000}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `image_result` | object `schema://nova/post-image/image-result@0.1.0` | `post_id`, `status`, `provider`, `preset`, `size`, `image_ref`, `content_hash`, `cost` | the post record, the user's channel and the run record | internal; becomes public only if the post is published later | dimensions, hash, attachment and usage row cross-checked (Section I) |

```json
{"$id":"schema://nova/post-image/image-result@0.1.0","type":"object","required":["post_id","status","provider","preset","size","image_ref","content_hash","cost"],
 "properties":{"post_id":{"type":"string"},"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},
 "provider":{"enum":["provider-a","provider-b"]},"model":{"type":"string"},"preset":{"enum":["linkedin-portrait","x-landscape","square"]},
 "size":{"type":"object","required":["w","h"],"properties":{"w":{"type":"integer"},"h":{"type":"integer"}}},"safe_zone":{"type":"object"},"headline":{"type":"string"},
 "image_ref":{"type":["string","null"]},"content_hash":{"type":["string","null"],"pattern":"^sha256:[0-9a-f]{64}$"},"prompt_ref":{"type":"string"},"usage_record_id":{"type":["string","null"]},
 "cost":{"type":"object","required":["amount","currency","basis"],"properties":{"amount":{"type":"number","minimum":0},"currency":{"type":"string"},"basis":{"enum":["provider_reported","estimate"]}}},
 "checks":{"type":"object","additionalProperties":{"enum":["pass","fail","not_run"]}}}}
```

Supporting schemas:

```json
{"$id":"schema://nova/post-image/post@0.1.0","type":"object","required":["post_id","tenant_id","platform","stage"],"properties":{"post_id":{"type":"string"},"tenant_id":{"type":"string"},"platform":{"enum":["linkedin","x","other"]},"stage":{"type":"string"},"hook":{"type":"string"},"body":{"type":"string"},"image_ref":{"type":["string","null"]},"image_prompt":{"type":["string","null"]},"image_status":{"enum":["none","ready","failed"]},"image_meta":{"type":["object","null"]},"image_history":{"type":"array","items":{"type":"string"}}}}
{"$id":"schema://nova/post-image/brand-kit@0.1.0","type":"object","required":["brand_kit_id","wordmark","palette"],"properties":{"brand_kit_id":{"type":"string"},"wordmark":{"type":"string","maxLength":40},"logomark_description":{"type":["string","null"]},"palette":{"type":"object","required":["background","accent","text"],"properties":{"background":{"type":"string","pattern":"^#[0-9A-Fa-f]{6}$"},"accent":{"type":"string","pattern":"^#[0-9A-Fa-f]{6}$"},"text":{"type":"string","pattern":"^#[0-9A-Fa-f]{6}$"}}}}}
{"$id":"schema://nova/post-image/provider-list@0.1.0","type":"object","required":["providers"],"properties":{"providers":{"type":"array","items":{"type":"object","required":["provider","ready","reason","sizes","price_per_image"],"properties":{"provider":{"enum":["provider-a","provider-b"]},"ready":{"type":"boolean"},"reason":{"type":"string"},"sizes":{"type":"array","items":{"type":"string"}},"price_per_image":{"type":"object","required":["amount","currency","as_of"],"properties":{"amount":{"type":"number","minimum":0},"currency":{"type":"string"},"as_of":{"type":"string","format":"date"}}}}}}}}
{"$id":"schema://nova/post-image/usage-record@0.1.0","type":"object","required":["usage_id","ts","provider","cost_usd","cost_basis","meta","idempotency_key"],"properties":{"usage_id":{"type":"string"},"ts":{"type":"string","format":"date-time"},"provider":{"type":"string"},"model":{"type":["string","null"]},"units":{"type":"object","properties":{"images":{"type":"integer","minimum":0},"size":{"type":"string"},"quality":{"type":["string","null"]}}},"cost_usd":{"type":"number"},"cost_basis":{"enum":["provider_reported","estimate","price_table"]},"meta":{"type":"object","required":["job"],"properties":{"job":{"const":"content:post-image"},"run_id":{"type":"string"},"adjusts_usage_id":{"type":["string","null"]}},"additionalProperties":{"type":["string","null"]}},"idempotency_key":{"type":"string"}}}
{"$id":"schema://nova/post-image/delivery@0.1.0","type":"object","required":["message_id","status"],"properties":{"message_id":{"type":"string"},"channel":{"type":"string"},"status":{"enum":["delivered","queued","failed"]},"delivered_at":{"type":["string","null"],"format":"date-time"}}}
```

**Example (fictional)**

Input: `{"post_id":"post_0042","platform_preset":"x-landscape","provider":"provider-b","quality":"high","brand_kit_ref":"brand_acme"}`

Output: `{"post_id":"post_0042","status":"completed","provider":"provider-b","model":"image-model-b","preset":"x-landscape","size":{"w":1536,"h":1024},"safe_zone":{"top_pct":12,"bottom_pct":12},"headline":"Smaller models, faster answers","image_ref":"asset://post-images/posts/post_0042/3b1f0c.png","content_hash":"sha256:example-hash-post-0042","prompt_ref":"run://run_0136/prompt","usage_record_id":"use_0981","cost":{"amount":0.08,"currency":"USD","basis":"estimate"},"checks":{"layers":"pass","safe_zone":"pass","dimensions":"pass","text_match":"not_run"}}` (prices are illustrative, not current vendor prices).

## E. Constraints and budgets

- **Hard constraints:** only the requesting user's own post; every word in the image comes from the post or the brand kit — no new numbers, claims or names; never draw real third-party logos, trademarked marks or copyrighted characters; no photoreal likeness of an identifiable real person; headline within the preset's word cap; text and logos inside the preset's safe zone; the provider is explicit and never switched silently; estimate ≤ per-image cap; ≤ 3 generations per post per run; a provider content-policy refusal is final for that prompt (no evasive rewording); never publish the post.
- **Soft constraints:** use the brand palette; tint the badge with the subject's accent colour when commenting on a third party; prefer fewer words.
- **Resource and tenant scope:** the requesting user's posts, brand kit, image bucket and ledger only.
- **Data sources allowed:** the post's hook and body, the brand kit, the user's request.
- **External transfers permitted:** the prompt text (no personal data beyond what the post already contains) to the chosen image provider only.
- **Time limit:** generation call ≤ 120 s; whole run ≤ 10 minutes.
- **Cost or token budget:** per-image cap and daily image budget `TBD` (operator). Stop if the estimate exceeds either.
- **Maximum steps:** 12 (S1–S12).
- **Maximum retries and correction cycles:** generation: 1 retry only on a transient error with no image returned; reads: 2 retries; prompt redraft: 1; total generations per post per run: 3.
- **Rate limits and concurrency:** 1 generation in flight per post; provider rate limits apply.
- **Retention and logging constraints:** prompts and images kept with the post; logs never contain provider keys or base64 image data.
- **Prohibited actions:** deleting the previous image; changing post text; switching provider without a new choice; regenerating beyond the budget.
- **Stop conditions:** post not found or not owned; no ready provider; estimate over cap; provider refusal; generation budget exhausted.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Image asset (one per accepted generation) | S9 checks passed | Allowed within the pre-approved cost scope | object exists at the hash path; hash matches |
| Create | Usage row | a generation call returned | Allowed | row readable by `usage_id` |
| Create | Image prompt | S5 verified | Allowed | stored with the run |
| Read | Post, brand kit, provider list | own tenant only | Allowed | — |
| Update | Post: `image_ref`, `image_prompt`, `image_status`, `image_meta`, `image_history` | post owned; stage not `published` | Allowed | read-back equals the new values |
| Delete | any (including the previous image) | — | Not Allowed | — |
| External side effect | paid call to the chosen image provider | estimate ≤ cap; budget left | Allowed within cap; Approval Required above cap or after a provider switch | provider response contains image data |
| External side effect | review message to the requesting user | run finished | Allowed | delivery receipt |

- **Idempotency:** generation key = hash(`run_id`, `post_id`, prompt hash, `provider`, attempt number); storage path = `posts/<post_id>/<content_hash>`; usage key = the generation key. A repeat with the same key reuses the stored result.
- **Expected side effects:** provider charges; a new object in the image bucket; the post's image fields change.
- **Compensation:** re-attach the previous image from `image_history` on request; the new object can be left unreferenced (no delete in this recipe).
- **Irreversible:** provider charges.
- **Ambiguous outcomes:** a timed-out generation is never retried blindly: the single permitted retry uses a new attempt number, and a usage row for the timed-out attempt is written only where the provider documents billing for it (`TBD` per provider), so the ledger stays truthful.

## G. Autonomy and approval policy

- **Default autonomy level:** A2 — bounded autonomy within the cost cap.
- **Planning level:** A2 (preset, provider eligibility, prompt).
- **Execution level:** A2 inside the pre-approved scope (estimate ≤ cap, chosen provider, ≤ 3 generations); A1 outside it.
- **Verification level:** A2.
- **Recovery level:** A1 — any provider switch or budget extension needs the user.
- **Approval required for:** a generation above the per-image cap; switching provider after a failure; a fourth generation for the same post.
- **Actions never permitted:** drawing third-party logos or a real person's likeness; adding facts to the image; publishing; deleting images; rewording to evade a provider refusal.
- **Conditions requiring clarification:** post has no platform or no text; preset contradicts the platform.
- **Conditions requiring escalation:** provider refusal; budget exhausted; repeated verification failure.
- **Safe state after stop, cancellation or escalation:** the post keeps its previous image; nothing deleted; spend reported.

Approval covers one generation with the stated provider, preset and estimate. It does not cover regenerations or another provider.

## H. Execution plan

**S1 — Validate the request**
1. Objective: a schema-valid request.
2. Capability: none (local validation).
3. Input mapping: user message or UI action → `request@0.1.0`.
4. Dependencies: none.
5. Preconditions: an explicit image request.
6. Action / side effects: parse and validate; no side effects.
7. Autonomy: A2; no approval.
8. Timeout / retry: 5 s / none.
9. Idempotency: not applicable.
10. Postconditions: valid request.
11. Verification: JSON Schema passes.
12. Failure routes: missing `post_id` → ask once → else `blocked`.
13. Escalation triggers: none.
14. Evidence: validated request.

**S2 — Load the post and brand kit**
1. Objective: the post text, platform and current image; the brand kit.
2. Capability: `nova.content.posts.get`; brand kit read.
3. Input mapping: `post_id`, `brand_kit_ref`.
4. Dependencies: S1.
5. Preconditions: request valid.
6. Action / side effects: read-only.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: post with non-empty text; preset resolved.
11. Verification: post tenant = requesting tenant; stage ≠ `published`.
12. Failure routes: not found or other tenant → `blocked`; published → `blocked` ("image changes after publishing are out of scope").
13. Escalation: none.
14. Evidence: post ID, stage, platform, preset.

**S3 — Provider readiness and estimate**
1. Objective: confirm the chosen provider can serve the preset within budget.
2. Capability: `nova.content.image.list_providers`.
3. Input mapping: `provider`, preset, `quality`.
4. Dependencies: S2.
5. Preconditions: preset known.
6. Action: read readiness, sizes and price; map the preset to the provider's size; compute the estimate and remaining daily budget.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 1 retry.
9. Idempotency: read-only.
10. Postconditions: `ready`, size, crop note, estimate.
11. Verification: size is one the provider lists; estimate ≤ min(per-image cap, remaining budget).
12. Failure routes: not ready → `blocked` with reason and the other variant's estimate; over cap → S6 approval.
13. Escalation: none.
14. Evidence: provider list snapshot with `as_of`.

**S4 — Compose the layered prompt**
1. Objective: an image prompt that follows the five layers and the preset.
2. Capability: `nova.content.image_prompt.compose.default`.
3. Input mapping: hook, body, preset, brand kit, `topic_badge`, `style_notes`.
4. Dependencies: S3.
5. Preconditions: provider eligible.
6. Action: distil the headline (word cap), pick ≤ 4 facts as icon rows, choose one motif, write the atmosphere and brand layers, add the exact size and safe-zone sentence.
7. Autonomy: A2.
8. Timeout / retry: 30 s / 1 retry.
9. Idempotency: not applicable.
10. Postconditions: `image-prompt@0.1.0`.
11. Verification: schema valid (checked fully in S5).
12. Failure routes: compose error → retry once → `failed`.
13. Escalation: none.
14. Evidence: prompt object.

**S5 — Verify the prompt**
1. Objective: prove the prompt obeys every rule before money is spent.
2. Capability: rule check (deterministic) plus a second LLM pass for fact support.
3. Input mapping: prompt, post text, preset.
4. Dependencies: S4.
5. Preconditions: prompt exists.
6. Action: check all five layers present; headline word count ≤ cap; rows ≤ preset limit with icons; no third-party logo or real-person likeness requested; every number and name appears in the post; safe-zone and size sentence present.
7. Autonomy: A2.
8. Timeout / retry: 15 s / none.
9. Idempotency: read-only.
10. Postconditions: all checks pass.
11. Verification: each rule records pass or fail.
12. Failure routes: any fail → redraft once (S4) → else `failed` with the failed rules.
13. Escalation: none.
14. Evidence: rule results.

**S6 — Cost checkpoint**
1. Objective: spend only within the approved scope.
2. Capability: user interaction (only when needed).
3. Input mapping: provider, size, quality, estimate, crop note, generations left.
4. Dependencies: S5.
5. Preconditions: prompt verified.
6. Action: within cap → proceed and disclose the estimate; above cap or after a switch → ask "Generate with provider B, 1536×1024, high, about 0.08 USD?" (Approve / Change / Cancel).
7. Autonomy: A2 within cap; A0 above it.
8. Timeout / retry: wait up to 30 minutes, then `cancelled`.
9. Idempotency: the approval ID binds one generation key.
10. Postconditions: proceed flag or approval record.
11. Verification: approval references the prompt hash and estimate.
12. Failure routes: Change → S3; Cancel or timeout → `cancelled`.
13. Escalation: not applicable.
14. Evidence: estimate, approval ID if any.

**S7 — Generate the image**
1. Objective: one image from the chosen provider.
2. Capability: `nova.content.image.generate.provider-a` or `.provider-b`.
3. Input mapping: `prompt_text`, size, quality, generation key.
4. Dependencies: S6.
5. Preconditions: within budget; provider still ready.
6. Action / side effects: paid provider call; returns base64 or a short-lived URL.
7. Autonomy: A2 (covered by S6).
8. Timeout / retry: 120 s / 1 retry on a transient error with no image.
9. Idempotency: generation key.
10. Postconditions: image bytes in memory, or a named error.
11. Verification: response decodes to an image.
12. Failure routes: content-policy refusal → `blocked` (no rewording); hard error → `failed`; no silent provider switch.
13. Escalation: refusal → tell the user.
14. Evidence: provider request ID, model, latency.

**S8 — Record the cost**
1. Objective: one truthful usage row per generation call.
2. Capability: `nova.usage.ledger.record`.
3. Input mapping: provider, model, size, quality, cost (provider-reported if present, else estimate), `meta.job = content:post-image`, `post_id`, `run_id`.
4. Dependencies: S7 returned (success or billed failure).
5. Preconditions: a generation call completed.
6. Action / side effects: append one row.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: usage key = generation key (unique).
10. Postconditions: row exists.
11. Verification: read back by `usage_id`; `cost_basis` set.
12. Failure routes: ledger down → keep the row in the run record and mark `partially_completed`.
13. Escalation: none.
14. Evidence: `usage_id`.

**S9 — Validate the output**
1. Objective: the file matches the preset.
2. Capability: local image inspection.
3. Input mapping: image bytes, expected size.
4. Dependencies: S7.
5. Preconditions: bytes present.
6. Action: decode; check format (PNG, JPEG or WebP), dimensions or aspect within ±2 % of the requested size, file ≤ storage limit, not blank; compute `sha256`.
7. Autonomy: A2.
8. Timeout / retry: 10 s / none.
9. Idempotency: read-only.
10. Postconditions: `content_hash`.
11. Verification: all checks pass.
12. Failure routes: wrong aspect or blank → counts as one generation; offer a regeneration within budget.
13. Escalation: none.
14. Evidence: dimensions, hash.

**S10 — Store the image**
1. Objective: durable storage before the provider link expires.
2. Capability: `nova.content.assets.upload`.
3. Input mapping: bytes, `posts/<post_id>/<content_hash>`.
4. Dependencies: S9.
5. Preconditions: output valid.
6. Action / side effects: create one object.
7. Autonomy: A2.
8. Timeout / retry: 30 s / 2 retries.
9. Idempotency: hash path (re-upload is a no-op).
10. Postconditions: object readable.
11. Verification: read back the object's hash.
12. Failure routes: storage fails → `partially_completed` (generated and paid, not stored); report.
13. Escalation: none.
14. Evidence: `image_ref`.

**S11 — Attach to the post**
1. Objective: the post points at the new image; the old one is kept.
2. Capability: `nova.content.posts.attach_image`.
3. Input mapping: `post_id`, `image_ref`, prompt, `image_meta` (provider, size, preset, hash).
4. Dependencies: S10.
5. Preconditions: post stage still ≠ `published`.
6. Action / side effects: update image fields; push the previous image onto `image_history`.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: conditional update on the previous `image_ref`.
10. Postconditions: post shows the new image.
11. Verification: read-back equals the new values.
12. Failure routes: conflict (post changed) → re-read, ask the user; else `partially_completed`.
13. Escalation: none.
14. Evidence: post version before and after.

**S12 — Independent check and deliver**
1. Objective: a truthful review package for the user.
2. Capability: separate verifier (Section K); `nova.notify.owner.send`.
3. Input mapping: `image_result`, image preview, prompt, cost.
4. Dependencies: S11.
5. Preconditions: image attached.
6. Action: run the independent check; send preview, headline, provider, size, crop note and cost; the user approves the image later with the post.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 2 retries for delivery.
9. Idempotency: message key = `run_id`.
10. Postconditions: delivered.
11. Verification: delivery receipt.
12. Failure routes: delivery fails → result stays in the run record; `partially_completed`.
13. Escalation: verifier finds a logo, a garbled headline or an unsupported fact → flag in the message, do not hide it.
14. Evidence: message ID, verifier result.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Input validity | request matches `request@0.1.0` | validator | valid | ask once, then `blocked` |
| Ownership | post tenant = requester's tenant | post record | equal | `blocked` |
| Provider explicit | variant used = variant chosen or approved | run log vs S6 | equal | stop; report |
| Layer rule | five layers present, word cap, row limit | S5 rule results | all pass | redraft once |
| Fact support | every number and name in the prompt appears in the post | S5 verifier | 100 % | redraft or `failed` |
| No third-party marks | no real logo or likeness requested | S5 rule | none | remove and redraft |
| Dimensions | output aspect within ±2 % of the preset size | S9 | within | regenerate within budget |
| Safe zone | prompt carries the safe-zone sentence; verifier sees no text in the margins | S5, Section K | pass | flag to the user |
| Storage | stored hash = computed hash | S10 | equal | retry, then `partially_completed` |
| Attachment | post `image_ref` = stored ref | S11 read-back | equal | report |
| Cost recorded | one usage row per generation call | ledger | count matches calls | report gap |
| Output schema | `image_result` valid | validator | valid | fix fields; never fake values |

A provider returning an image proves only that an image exists, not that its text is legible or inside the safe zone.

## J. Self-correction and recovery

- **Failure classes:** validation, missing post, provider not ready, over budget, prompt-rule failure, content-policy refusal, transient provider error, wrong dimensions, storage failure, update conflict.
- **Permitted corrections:** redraft the prompt once; one retry of a transient generation error; regenerate within the 3-per-post budget after a dimension or legibility failure; retry reads, storage and ledger writes (bounded).
- **Forbidden corrections:** switching provider without a new choice; rewording to slip past a refusal; dropping the safe zone or word cap to make a render pass; deleting the previous image; recording a cost of zero to hide spend.
- **Maximum correction cycles:** 1 per step; 3 generations per post per run.
- **Reverification required after correction:** yes — S5 after any redraft; S9 after any regeneration.
- **Stop conditions:** budget exhausted; refusal; hard constraint fails.
- **Recovery or compensation:** re-attach the previous image from history on request.
- **What must be reported:** every generation (and its cost), every failed rule, provider refusals, and any image stored but not attached.

## K. Independent challenge

- **Assumptions to challenge:** the headline in the image matches the intended text; nothing important falls outside the platform crop; the badge does not imply endorsement by a third party.
- **Alternative approach:** would the post work better with no image, or with the user's own photo? Mention it when two generations have failed.
- **Contradictory evidence:** rendered text that differs from the headline; a shape resembling a real logo.
- **Could mislead:** an image claiming numbers the post does not contain; a crop that hides the headline in the feed.
- **Separate verifier:** a vision-capable model, distinct from the generator, reads the rendered text and compares it with the headline, checks the margins for text and looks for logo-like marks; the user also reviews the image before publishing.
- **Findings requiring disclosure:** garbled or misspelled text, content near the crop edge, crop mapping (for example 2:3 output for a 4:5 slot).

## L. Human escalation

- **Triggers:** estimate over cap; chosen provider not ready (offer the other with its estimate); content-policy refusal; budget exhausted; verifier finds a logo-like mark or garbled headline.
- **Why the agent cannot decide:** money, brand risk and taste.
- **Information presented:** post excerpt, preset, provider options with estimates and crop notes, prompt, failed rules, image preview.
- **Decision requested:** approve the spend; choose a provider; accept, regenerate or drop the image.
- **Options and consequences:** stated per trigger (for example, provider A costs more per image but renders dense text more reliably).
- **Safe behaviour while waiting:** no generation; the post keeps its previous image.
- **If no response:** after 30 minutes the run ends `cancelled` (nothing generated) or `partially_completed` (image generated, user review pending).
- **Resume:** a new choice re-runs S3 readiness and S5 before generating.

## M. Completion conditions

Mandatory: `image_result` valid; the provider used was explicit; S5 rules passed; output dimensions match the preset; image stored and attached; one usage row per generation call; limitations disclosed; evidence references present.

- `completed` — image generated, verified, stored, attached, cost recorded and review package delivered.
- `partially_completed` — image generated and paid but storing, attaching, cost recording or delivery failed, or the verifier flagged an issue the user must review.
- `blocked` — invalid input, post not owned or published, provider not ready, over cap without approval, or provider refusal.
- `failed` — generation or prompt verification failed and recovery was exhausted.
- `cancelled` — the user declined or approval timed out.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome; `image_result`; step statuses S1–S12; verification results and failed checks; evidence references (prompt, provider request ID, `image_ref`, `usage_id`); approvals and scope; side effects (charges, stored object, post update) and reconciliation status; corrections and regenerations; unresolved challenges; limitations (crop mapping, verifier not run); escalation state; learning candidate. Never include provider keys or raw base64 image data.

## O. Learning and revision

- **Outcome metrics:** first-attempt acceptance rate per provider and preset, generations per accepted image, cost per accepted image, prompt-rule failure rate, verifier flag rate.
- **Evidence retained:** prompt, provider, size, hash, usage row, user verdict.
- **Data minimisation:** metrics stored without post text; rejected images unreferenced and removed by the operator's storage retention policy (`TBD`).
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}` (for example a layer wording change that raised legibility).
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review; no silent changes to the layer rule, presets or budgets.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (test post: provider B `x-landscape`; provider A `linkedin-portrait`) | correctly sized image with safe-zone prompt, stored, attached, one usage row each | UNVERIFIED on Skill Port; observed in the reference implementation |
| Missing input (`post_id` absent) | asks once, then `blocked` | TBD |
| Invalid input (unknown preset, `max_cost_usd` = 0) | rejected at S1 | TBD |
| Missing capability (provider binding absent) | `blocked` with a dependency message | TBD |
| Provider not ready | `blocked`; other variant offered with its estimate; no silent switch | TBD |
| Permission denied (`image:generate:provider-b` missing) | `blocked`, no call | TBD |
| Unauthorized scope (another tenant's post) | denied at S2 | TBD |
| Tenant isolation | cannot read another tenant's brand kit or images | TBD |
| Timeout on generation | one retry with a new attempt number; ledger truthful | TBD |
| Duplicate / replay (same generation key) | stored result reused; no second charge | TBD |
| Prompt rule failure (headline over cap, third-party logo requested) | redraft once, then `failed` | TBD |
| False success (provider 200 without image data, or wrong dimensions) | not stored or attached; detected at S7/S9; regeneration offered within budget | TBD |
| Correction budget exhausted (3 generations) | stops; approval needed for a fourth | TBD |
| Escalation (estimate over cap) | asks; nothing generated while waiting | TBD |
| Cancellation (user cancels at S6) | `cancelled`; nothing generated | TBD |
| Compensation (user rejects the image) | previous image re-attached from history | TBD |
| Output schema and status | valid `image_result`, correct status | TBD |
| Compatibility / regression (provider model version change) | sizes and presets still map; tests pass | TBD |

All tests use test posts and a test brand kit in a sandbox tenant; never real customer content. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, schema registry and approval UI — TBD (owner: publisher, with Skill Port).
- Per-image cap, daily image budget and provider price table — TBD (owner: operator; prices change and must be date-stamped).
- Provider B has no native 4:5 or 16:9 size; its 2:3 and 3:2 outputs rely on the safe-zone rule to survive platform crops.
- Image generators can misspell rendered text; the vision verifier (Section K) and the user's review mitigate this.
- Whether a timed-out generation is billed differs by provider — TBD per provider.
- Platform crop behaviour changes over time; presets need periodic review (owner: publisher).
- Retention of rejected images — TBD (owner: operator).
