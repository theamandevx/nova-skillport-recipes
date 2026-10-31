# Daily AI topic brief

## A. Identity and ownership

- **Recipe ID:** `nova.knowledge.daily-ai-topic-brief` (proposed until assigned by Skill Port)
- **Name:** Daily AI topic brief
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** research
- **Tags:** daily-brief, scheduled, trending-topics, sources, owner-chat, model-fallback, failure-alerts, reliability · Vedaxi task type: none of the seven supported today maps directly
- **Summary:** Every morning at the owner's chosen time, the agent takes the freshest trending items for the owner's subject (AI by default), picks the top five, writes one sourced line for each and delivers the brief to the owner's chat. The scheduled job runs with a declared fallback model on a **different provider**, and any failure is reported to the owner instead of passing silently.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** The trending-topics radar (`nova.knowledge.trending-topics-radar`) or an equivalent topic store; an HTTP fetcher; two model bindings on different providers (primary and fallback); an owner chat channel plus an optional secondary alert channel; a job-run registry with an independent missed-run watcher. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — a daily morning job that refreshes the topic feed and delivers a top-5 digest with links to the owner's chat runs in Nova's reference implementation. The cross-provider fallback and failure reporting specified here have not yet been verified. Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.ops.providers` | Capability | Report whether a model provider binding can serve a call now | low | UNVERIFIED |
| `nova.ops.providers.check` | Operation (read) | Reachability, authentication and credit/quota status where the provider exposes it | low | UNVERIFIED |
| `nova.topics.feed` | Capability | Get fresh trending items for a subject | low | UNVERIFIED |
| `nova.topics.feed.refresh` | Operation (create, via the radar recipe) | Run the radar on demand when stored items are stale | low | UNVERIFIED |
| `nova.topics.store` | Capability | Read topic candidates and record their status | low | UNVERIFIED |
| `nova.topics.store.list` | Operation (read) | List stored candidates by subject, status and age | low | UNVERIFIED |
| `nova.topics.store.set_status` | Operation (update) | Mark items `sent` in a brief, or `used` on the owner's pick | low | UNVERIFIED |
| `nova.research.fetch` | Capability | Read a public web page as plain text | low | UNVERIFIED |
| `nova.research.fetch.get` | Operation (read) | Read an item's page to ground its one-line summary | low | UNVERIFIED |
| `nova.brief.compose` | Capability | Write the brief text from selected items | medium | UNVERIFIED |
| `nova.brief.compose.write` | Operation (read, LLM) | One line per item, from the item's title, snippet and fetched text only | medium | UNVERIFIED |
| `nova.brief.verify` | Capability | Check the brief before delivery | low | UNVERIFIED |
| `nova.brief.verify.check` | Operation (read) | Item count, links, grounding of names and numbers, length | low | UNVERIFIED |
| `nova.notify.owner` | Capability | Deliver messages to the owner | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Send the brief or a failure alert | low | UNVERIFIED |
| `nova.ops.jobs` | Capability | Make run outcomes and missed runs visible | low | UNVERIFIED |
| `nova.ops.jobs.heartbeat` | Operation (create) | Record one run's status, model used and failures | low | UNVERIFIED |
| `nova.ops.jobs.watch` | Operation (read + alert) | Independently detect a missing or failed run after its deadline | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.brief.compose.write.primary` — the owner's chosen tier (default **medium**) on provider A.
- `nova.brief.compose.write.fallback` — **mandatory**; the same tier on **provider B**, where provider B is a different company with a separate account and credit balance. A second model behind the same router, aggregator or billing account does **not** count — it shares the outage the fallback exists to survive. Installation is refused when the fallback is missing or shares a provider or billing account with the primary. A fallback on a cheaper tier is allowed only if the owner approves it at installation, and every brief written by it says so.
- **Selection policy:** primary first; fallback once, only on a provider-side failure (authentication, credit or quota exhausted, rate limit after one retry, 5xx after one retry, timeout, empty completion). The fallback is never used to get a "better" answer and never for content or policy errors. Every brief records `model_used` and says in its footer when the fallback wrote it — the switch is never silent.
- `nova.brief.compose.write.headlines-only` — deterministic degraded output, **not a model**: when both providers fail, the five headlines with links are sent from a fixed template, labelled "AI summaries unavailable today", together with a failure alert.
- `nova.notify.owner.send.default` — the owner's chat (for example Telegram). `nova.notify.owner.send.template-alert` — failure alerts from a fixed template with no LLM dependency, so an LLM outage can still be reported. `nova.notify.owner.send.secondary-channel` — optional second channel (for example email to the owner's own address); eligible for failure alerts only, and only when the chat channel itself fails.
- `nova.ops.jobs.watch.default` — a separate scheduled check, independent of the brief job and of both model providers.
- All other operations: only the default variant is supported.

**Bindings:** `binding://nova/ops/provider-check`, `binding://nova/topics/refresh`, `binding://nova/topics/store-list`, `binding://nova/topics/store-status`, `binding://nova/research/fetch`, `binding://nova/brief/write-primary`, `binding://nova/brief/write-fallback`, `binding://nova/brief/verify`, `binding://nova/notify/owner`, `binding://nova/notify/owner-secondary`, `binding://nova/ops/jobs-heartbeat`, `binding://nova/ops/jobs-watch` — all `status: UNVERIFIED`. Reference implementation: an agent framework's built-in cron job (daily, morning, owner's time zone) that runs a content skill and delivers to the owner's Telegram chat; topics come from a nightly n8n feed workflow; a separate job registry discovers recurring jobs every few minutes and labels stale runs honestly.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Brief config | `schema://nova/daily-ai-topic-brief/config@0.1.0` | the owner | read |
| Topic candidate | `schema://nova/trending-topics-radar/topic@0.1.0` | the owner's tenant | read, update (`status`, `sent_in_brief_at`) |
| Brief | `schema://nova/daily-ai-topic-brief/brief-result@0.1.0` | the owner's channel and run record | create |
| Failure alert | `schema://nova/daily-ai-topic-brief/failure-alert@0.1.0` | the owner's channel | create |
| Run record | `schema://nova/daily-ai-topic-brief/run-record@0.1.0` | the job registry | create |

**Authorization requirements (declared, never a grant):** `topics:read`, `topics:write` (status fields only), `topics:refresh`, `web:fetch:public`, `llm:invoke:primary`, `llm:invoke:fallback`, `providers:health:read`, `notify:owner`, `notify:owner:secondary` (optional), `jobs:heartbeat`, `jobs:watch`, and `schedule:create` at installation (owner approval). Keys as `secret_ref://llm-provider-a` and `secret_ref://llm-provider-b` — two separate accounts.

**Policy / trust:** risk class `low` for content (public items, owner-only delivery) with a **reliability requirement**: a failed or missed run must reach the owner. Fetched pages and titles are untrusted data.

## C. Intent and applicability

- **User problem:** The owner wants a dependable morning snapshot of what is new in their subject, with links, without opening several sites — and needs to know when it did not arrive and why.
- **Desired outcome:** Every scheduled morning, either a five-item sourced brief in the owner's chat, or a clear failure alert stating what broke and what to do.
- **Trigger or entry conditions:** The installed daily schedule fires (default 07:00 in the owner's time zone); the owner may also ask "send today's brief now".
- **Applicable situations:** Daily awareness of a field; picking a topic for a post (the owner replies with an item number).
- **Out-of-scope situations:** Monitoring named companies (competitor monitor recipe); writing posts (content recipes); briefs to anyone other than the owner; market or investment advice.
- **Required inputs:** a brief config with subject, send time, time zone and both model variants (see Section D).
- **Prerequisites:** an owner-approved daily job **and** an independent watcher job; two model providers with separate accounts; an owner chat channel; a topic source (the radar recipe or equivalent).
- **Required capabilities:** `nova.ops.providers`, `nova.topics.feed`, `nova.topics.store`, `nova.brief.compose`, `nova.brief.verify`, `nova.notify.owner`, `nova.ops.jobs`; optional `nova.research.fetch` (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port.
- **Assumptions:** Verified in the reference implementation: the daily job, feed refresh, top-5 digest and chat delivery. Assumed: the target runtime lets a scheduled job call two independent model bindings and send a template message without an LLM.
- **Ambiguity policy:** Stop at installation until both model variants and the owner channel are configured; at run time use documented safe defaults. Never send an invented item.
- **Vedaxi task type:** None maps directly.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `subject_keys` | array of radar topic keys | yes | owner config | `["ai"]` | Unknown key: `blocked` at installation. |
| `send_time` | `HH:MM` | no | owner config | `07:00` | Invalid: `blocked` at installation. |
| `timezone` | IANA time-zone name | yes | owner config | — | Invalid: `blocked`. |
| `items` | integer 3–7 | no | owner config | 5 | Clamp. |
| `read_sources` | boolean | no | owner config | `true` | — |
| `primary` | `{tier, provider_ref}` | yes | operator | tier `medium` | Missing: installation refused. |
| `fallback` | `{tier, provider_ref, billing_account_ref}` | yes | operator | same tier as primary | Missing, same provider or same billing account: installation refused. |
| `max_chars` | integer 800–3,800 | no | channel limit | 3,500 | Clamp to the channel limit. |
| `repeat_window_days` | integer 1–30 | no | owner config | 7 | Clamp. |
| `secondary_alert_channel` | channel reference (for example `jane@example.com`) | no | owner; personal data | none | Absent: alerts use the chat only; disclosed at installation. |

Reused schemas (acknowledged): `topic@0.1.0`, `radar-result@0.1.0` and `config@0.1.0` are defined in Section D of `nova.knowledge.trending-topics-radar` (folder `30-trending-topics-radar`) and used here unchanged. Own schemas:

```json
{"$id":"schema://nova/daily-ai-topic-brief/config@0.1.0","type":"object","required":["subject_keys","timezone","primary","fallback"],
 "properties":{"subject_keys":{"type":"array","minItems":1,"items":{"type":"string"}},
 "send_time":{"type":"string","pattern":"^([01][0-9]|2[0-3]):[0-5][0-9]$"},"timezone":{"type":"string"},
 "items":{"type":"integer","minimum":3,"maximum":7},"read_sources":{"type":"boolean"},
 "primary":{"type":"object","required":["tier","provider_ref"],"properties":{"tier":{"enum":["cheap","medium","pro"]},"provider_ref":{"type":"string"}}},
 "fallback":{"type":"object","required":["tier","provider_ref","billing_account_ref"],
  "properties":{"tier":{"enum":["cheap","medium","pro"]},"provider_ref":{"type":"string"},"billing_account_ref":{"type":"string"}}},
 "max_chars":{"type":"integer","minimum":800,"maximum":3800},"repeat_window_days":{"type":"integer","minimum":1,"maximum":30},
 "secondary_alert_channel":{"type":"string"}}}
```

Installation rule (outside JSON Schema): `fallback.provider_ref ≠ primary.provider_ref` **and** `fallback.billing_account_ref` differs from the primary's account; a lower fallback tier needs a recorded owner approval.

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `brief_result` | `schema://nova/daily-ai-topic-brief/brief-result@0.1.0` | `run_id`, `date`, `status`, `items`, `model_used`, `fallback_used`, `delivered` | owner chat; run record | low | S6 checks; delivery receipt |
| `failure_alert` | `schema://nova/daily-ai-topic-brief/failure-alert@0.1.0` | `job`, `date`, `stage`, `reason`, `next_run_at` | owner chat, else secondary channel | low | delivery receipt |

```json
{"$id":"schema://nova/daily-ai-topic-brief/brief-result@0.1.0","type":"object",
 "required":["run_id","date","status","items","model_used","fallback_used","delivered"],
 "properties":{"run_id":{"type":"string"},"date":{"type":"string","format":"date"},
 "status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},
 "items":{"type":"array","maxItems":7,"items":{"type":"object","required":["rank","title","url","line_basis"],
  "properties":{"rank":{"type":"integer","minimum":1},"topic_id":{"type":"string"},"title":{"type":"string"},
  "url":{"type":"string","format":"uri"},"line":{"type":["string","null"],"maxLength":240},
  "line_basis":{"enum":["fetched_page","headline_only"]}}}},
 "model_used":{"enum":["primary","fallback","none"]},"fallback_used":{"type":"boolean"},"degraded":{"type":"boolean"},
 "feed_age_hours":{"type":"number"},"delivered":{"type":"boolean"},"message_id":{"type":["string","null"]},
 "failure":{"type":["object","null"],"properties":{"stage":{"type":"string"},"reason":{"type":"string"},
  "providers_tried":{"type":"array","items":{"type":"string"}}}}}}
```

```json
{"$id":"schema://nova/daily-ai-topic-brief/failure-alert@0.1.0","type":"object","required":["job","date","stage","reason","next_run_at"],
 "properties":{"job":{"type":"string"},"date":{"type":"string","format":"date"},
 "stage":{"enum":["preflight","feed","compose","verify","deliver","missed_run"]},"reason":{"type":"string","maxLength":300},
 "providers_tried":{"type":"array","items":{"type":"string"}},"consecutive_failures":{"type":"integer","minimum":1},
 "next_run_at":{"type":"string","format":"date-time"},"suggested_action":{"type":"string","maxLength":200}}}
```

Supporting schemas (compact, one per line):

```json
{"$id":"schema://nova/daily-ai-topic-brief/provider-health@0.1.0","type":"object","required":["provider_ref","reachable","auth_ok","credit_status","checked_at"],"properties":{"provider_ref":{"type":"string"},"reachable":{"type":"boolean"},"auth_ok":{"type":"boolean"},"credit_status":{"enum":["ok","low","exhausted","unknown"]},"checked_at":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/daily-ai-topic-brief/topic-query@0.1.0","type":"object","required":["subject_keys"],"properties":{"subject_keys":{"type":"array","minItems":1,"items":{"type":"string"}},"status":{"enum":["new","sent","used","dismissed"]},"max_age_hours":{"type":"integer","minimum":1,"maximum":72},"exclude_sent_within_days":{"type":"integer","minimum":0,"maximum":30},"limit":{"type":"integer","minimum":1,"maximum":200}}}
{"$id":"schema://nova/daily-ai-topic-brief/topic-status@0.1.0","type":"object","required":["topic_id","status"],"properties":{"topic_id":{"type":"string"},"status":{"enum":["sent","used"]},"sent_in_brief_at":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/daily-ai-topic-brief/fetch-input@0.1.0","type":"object","required":["url"],"properties":{"url":{"type":"string","format":"uri","pattern":"^https?://"},"max_bytes":{"type":"integer","minimum":1,"maximum":2097152}}}
{"$id":"schema://nova/daily-ai-topic-brief/page@0.1.0","type":"object","required":["url","status","usable"],"properties":{"url":{"type":"string","format":"uri"},"final_url":{"type":"string","format":"uri"},"status":{"type":"integer"},"text":{"type":["string","null"]},"fetched_at":{"type":"string","format":"date-time"},"content_hash":{"type":"string"},"usable":{"type":"boolean"}}}
{"$id":"schema://nova/daily-ai-topic-brief/compose-input@0.1.0","type":"object","required":["items","max_chars","variant"],"properties":{"date":{"type":"string","format":"date"},"items":{"type":"array","minItems":1,"maxItems":7,"items":{"type":"object","required":["rank","title","url"],"properties":{"rank":{"type":"integer","minimum":1},"topic_id":{"type":"string"},"title":{"type":"string"},"url":{"type":"string","format":"uri"},"snippet":{"type":"string"},"page_text":{"type":["string","null"]}}}},"max_chars":{"type":"integer","minimum":800,"maximum":3800},"variant":{"enum":["primary","fallback","headlines-only"]}}}
{"$id":"schema://nova/daily-ai-topic-brief/verify-report@0.1.0","type":"object","required":["passed","checks"],"properties":{"passed":{"type":"boolean"},"checks":{"type":"object","properties":{"item_count":{"type":"boolean"},"links":{"type":"boolean"},"grounding":{"type":"boolean"},"length":{"type":"boolean"},"footer":{"type":"boolean"}}},"corrections":{"type":"array","items":{"type":"string"}}}}
{"$id":"schema://nova/daily-ai-topic-brief/delivery@0.1.0","type":"object","required":["delivered"],"properties":{"message_id":{"type":["string","null"]},"channel":{"enum":["owner_chat","secondary"]},"delivered":{"type":"boolean"},"delivered_at":{"type":["string","null"],"format":"date-time"},"error":{"type":["string","null"]}}}
{"$id":"schema://nova/daily-ai-topic-brief/run-record@0.1.0","type":"object","required":["job_id","run_id","status"],"properties":{"job_id":{"type":"string"},"run_id":{"type":"string"},"scheduled_for":{"type":"string","format":"date-time"},"started_at":{"type":"string","format":"date-time"},"finished_at":{"type":["string","null"],"format":"date-time"},"status":{"enum":["completed","partially_completed","blocked","failed","cancelled","failed_unreported"]},"counts":{"type":"object","additionalProperties":{"type":"integer"}},"errors":{"type":"array","items":{"type":"string"}},"model_used":{"enum":["primary","fallback","none"]},"fallback_used":{"type":"boolean"},"failure_stage":{"type":["string","null"]},"consecutive_failures":{"type":"integer","minimum":0}}}
```

**Example (fictional)**

Delivered brief (fallback used): "Your AI brief — Fri 31 Oct · 1) Open-source framework adds parallel tool calling — the release lets agents call several tools at once. https://example.com/news/parallel-tool-calling · 2) … · Reply with a number to draft a post. — Written by the fallback model (provider B): the primary returned 'credit exhausted'."

`brief_result`: `{"run_id":"run_0902","date":"2026-10-31","status":"completed","items":[{"rank":1,"topic_id":"tp_0412","title":"Open-source framework adds parallel tool calling","url":"https://example.com/news/parallel-tool-calling","line":"The release lets agents call several tools at once.","line_basis":"fetched_page"}],"model_used":"fallback","fallback_used":true,"degraded":false,"feed_age_hours":5.0,"delivered":true,"message_id":"msg_7781","failure":{"stage":"compose","reason":"primary: credit exhausted","providers_tried":["provider-a","provider-b"]}}`

Failure alert (both providers down): `{"job":"daily-ai-topic-brief","date":"2026-10-31","stage":"compose","reason":"provider A: credit exhausted; provider B: authentication failed","providers_tried":["provider-a","provider-b"],"consecutive_failures":2,"next_run_at":"2026-11-01T07:00:00-04:00","suggested_action":"Top up provider A credit or renew the provider B key."}`

## E. Constraints and budgets

- **Hard constraints:** a fallback on a different provider and billing account must exist before the job can activate; every run ends in exactly one of — a delivered brief, a delivered degraded brief plus alert, or a delivered failure alert — never in silence; failure alerts use a template with no LLM dependency; every item links to a URL from the topic store (never generated); lines add no names or numbers absent from the title, snippet or fetched page; delivery to the owner only; the model that wrote the brief is always recorded and a fallback is stated in the message.
- **Soft constraints:** at most 2 items per site; skip items sent in the last `repeat_window_days`; one line per item ≤ 240 characters.
- **Resource and tenant scope:** the owner's topic store, config and channels.
- **Data sources allowed:** the topic store (refreshed by the radar recipe) and the items' own pages.
- **External transfers permitted:** item titles, snippets and page text to the provider that writes the brief; page requests to source sites.
- **Time limit:** brief delivered within 20 minutes of `send_time`; a run still unfinished at +30 minutes is treated as failed by the watcher.
- **Cost or token budget:** per-run token cap `TBD`; monthly cap per provider `TBD` (operator); provider credit is checked in pre-flight where the provider exposes it.
- **Maximum steps:** 10 (S1–S10).
- **Maximum retries and correction cycles:** primary 1 retry for transient errors, then fallback once; fetch 1 retry per page; delivery 2 retries; correction cycles 1.
- **Rate limits and concurrency:** one brief per owner per day (a manual "send now" counts as that day's brief unless the owner asks for a second); one run at a time.
- **Retention and logging constraints:** run records kept `TBD` days (proposed 90); logs record provider names, error classes and counts — never keys or page text.
- **Prohibited actions:** a silent model or provider switch; suppressing a failure; resending the same brief; inventing items.
- **Stop conditions:** both providers and the degraded template path fail; owner channel and secondary channel both fail (the run record goes red and the watcher repeats the alert next window).

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Brief message, degraded brief or failure alert to the owner | one brief per day; alerts as needed | Allowed (owner only) | delivery receipt |
| Create | Topic candidates via `refresh` | under the radar recipe's own rules | Allowed | radar result |
| Create | Run record | every run, every path | Allowed | read back |
| Create | Daily job and watcher job | at installation; fallback rule satisfied | Approval Required (owner, once) | both jobs listed |
| Read | topic store; public pages; provider health | own tenant; public pages | Allowed | — |
| Update | Topic `status` → `sent` with `sent_in_brief_at`; `used` on the owner's pick | items in this brief only | Allowed | read back |
| Delete | any | — | Not Allowed | — |
| External side effect | message to the owner's chat or secondary channel | owner channels only | Allowed | receipt |

- **Idempotency:** brief key = (`job_id`, local date); an alert key = (`job_id`, local date, stage). A retried send checks for a receipt first.
- **Expected side effects:** one or two owner messages per day; provider usage; topic status updates.
- **Compensation:** a wrong brief cannot be unsent; a correction message may follow, with the owner told why.
- **Irreversible:** delivered messages; provider charges.
- **Ambiguous outcomes:** delivery timeout → query the channel by message key before resending; never send a duplicate brief.

## G. Autonomy and approval policy

- **Default autonomy level:** A2 — bounded autonomy (scheduled, owner-only output).
- **Planning level:** A2.
- **Execution level:** A2, including the declared fallback (pre-approved at installation).
- **Verification level:** A2.
- **Recovery level:** A2 within limits.
- **Approval required for:** installing or changing the schedule; choosing or changing the primary or fallback provider; a lower-tier fallback.
- **Actions never permitted:** using an undeclared model or provider; a silent switch; suppressing failures; messaging anyone other than the owner.
- **Conditions requiring clarification:** missing fallback or channel at installation.
- **Conditions requiring escalation:** 2 consecutive failed runs; credit `low` or `exhausted` on either provider in pre-flight; both channels failing.
- **Safe state after stop, cancellation or escalation:** no partial brief sent, run record written, alert delivered or queued for the watcher.

## H. Execution plan

**S1 — Pre-flight**
1. Objective: know before writing whether the run can succeed, and through which variant.
2. Capability: `nova.ops.providers.check.default`; local config validation.
3. Input mapping: config → `config@0.1.0`; provider references for primary and fallback.
4. Dependencies: none.
5. Preconditions: trigger from the approved schedule or the owner.
6. Action / side effects: validate config; confirm the fallback rule; check both providers (reachable, authenticated, credit status where exposed) and the owner channel.
7. Autonomy: A2.
8. Timeout / retry: 20 s per check / 1 retry.
9. Idempotency: read-only.
10. Postconditions: a plan — primary, fallback-only, or degraded — with reasons.
11. Verification: fallback provider and billing account differ from the primary's.
12. Failure routes: fallback rule violated → `blocked` and alert; both providers down → plan `degraded`; credit `low` → continue and add a warning to the brief.
13. Escalation triggers: credit `low`/`exhausted` on any provider.
14. Evidence: provider health records (no keys).

**S2 — Get fresh topics**
1. Objective: current items for the subject.
2. Capability: `nova.topics.store.list.default`; `nova.topics.feed.refresh.default` when stale.
3. Input mapping: `subject_keys`, status `new`, age.
4. Dependencies: S1.
5. Preconditions: topic store reachable.
6. Action / side effects: if the radar's last successful run is ≥ 24 h old, trigger a refresh; read items.
7. Autonomy: A2.
8. Timeout / retry: refresh ≤ 5 minutes / 1 retry; list 10 s / 2 retries.
9. Idempotency: list is read-only; refresh follows the radar's dedupe rules.
10. Postconditions: candidate items with `feed_age_hours`.
11. Verification: every candidate has a URL; feed age ≤ 36 h.
12. Failure routes: refresh fails → use items ≤ 36 h old and disclose; none → failure alert (`stage = feed`).
13. Escalation triggers: none.
14. Evidence: feed age, candidate count.

**S3 — Select the items**
1. Objective: the top `items` (default 5).
2. Capability: none (local deterministic selection).
3. Input mapping: candidates, `repeat_window_days`.
4. Dependencies: S2.
5. Preconditions: candidates present.
6. Action / side effects: sort by score; drop items sent in the window; at most 2 per site.
7. Autonomy: A2.
8. Timeout / retry: 2 s / none.
9. Idempotency: deterministic.
10. Postconditions: 1–`items` items; fewer than 3 is disclosed in the brief.
11. Verification: no repeats within the window; per-site cap respected.
12. Failure routes: zero items → failure alert (`stage = feed`, reason "no new items").
13. Escalation triggers: none.
14. Evidence: selected topic IDs.

**S4 — Read the sources** (when `read_sources = true`)
1. Objective: page text to ground each line.
2. Capability: `nova.research.fetch.get.default`.
3. Input mapping: item URLs.
4. Dependencies: S3.
5. Preconditions: public URLs.
6. Action / side effects: HTTP GET to the source sites.
7. Autonomy: A2.
8. Timeout / retry: 15 s per page / 1 retry.
9. Idempotency: read-only.
10. Postconditions: text per item, or `headline_only`.
11. Verification: text ≥ 500 characters and on topic.
12. Failure routes: unreadable page → that item becomes `headline_only`.
13. Escalation triggers: none.
14. Evidence: fetch statuses.

**S5 — Write the brief**
1. Objective: one grounded line per item.
2. Capability: `nova.brief.compose.write.primary`, then `.fallback`, then `.headlines-only`.
3. Input mapping: items with title, snippet, page text (as data), `max_chars`.
4. Dependencies: S4 (or S3).
5. Preconditions: plan from S1.
6. Action / side effects: item text sent to the writing provider.
7. Autonomy: A2 (fallback pre-approved at installation).
8. Timeout / retry: 45 s; primary 1 retry on transient errors → fallback once on provider-side failure → headlines-only template.
9. Idempotency: not applicable.
10. Postconditions: brief text with `model_used`, `fallback_used`, `degraded`.
11. Verification: deferred to S6; an empty completion counts as a provider failure.
12. Failure routes: both providers fail → degraded brief plus failure alert (`stage = compose`).
13. Escalation triggers: fallback used two days running → owner alert about the primary.
14. Evidence: variant used, error classes per provider.

**S6 — Verify the brief**
1. Objective: nothing ungrounded or broken is delivered.
2. Capability: `nova.brief.verify.check.default`.
3. Input mapping: brief, items, page texts.
4. Dependencies: S5.
5. Preconditions: brief present.
6. Action / side effects: check item count, that every link equals a selected item URL, that names and numbers in each line appear in its title, snippet or page, length ≤ `max_chars`, and the fallback footer is present when used.
7. Autonomy: A2.
8. Timeout / retry: 5 s / none.
9. Idempotency: deterministic.
10. Postconditions: verified brief.
11. Verification: 100 % of checks pass.
12. Failure routes: an ungrounded line → replace it with the headline only (`line_basis = headline_only`); a missing footer → add it; a broken link → drop the item and disclose.
13. Escalation triggers: none.
14. Evidence: check results.

**S7 — Deliver to the owner**
1. Objective: the brief in the owner's chat at the scheduled time.
2. Capability: `nova.notify.owner.send.default`.
3. Input mapping: verified brief; brief key.
4. Dependencies: S6.
5. Preconditions: no brief already delivered today (unless the owner asked for another).
6. Action / side effects: one message to the owner.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 2 retries after checking for a receipt.
9. Idempotency: brief key.
10. Postconditions: delivered once; included topics marked `sent`.
11. Verification: delivery receipt.
12. Failure routes: chat fails → failure alert via `secondary-channel` (`stage = deliver`).
13. Escalation triggers: both channels fail.
14. Evidence: message ID.

**S8 — Handle the owner's pick** (asynchronous)
1. Objective: turn a reply into a chosen topic.
2. Capability: `nova.topics.store.set_status.default`.
3. Input mapping: reply number → topic ID from today's brief.
4. Dependencies: S7.
5. Preconditions: reply from the owner's verified channel.
6. Action / side effects: topic marked `used`; handed to a drafting recipe if the owner asks.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: same status twice is a no-op.
10. Postconditions: status `used`.
11. Verification: read back.
12. Failure routes: unclear reply → ask which item.
13. Escalation triggers: none.
14. Evidence: topic ID.

**S9 — Record the outcome and report any failure**
1. Objective: no silent run.
2. Capability: `nova.ops.jobs.heartbeat.default`; `nova.notify.owner.send.template-alert` on failure.
3. Input mapping: status, `model_used`, failure stage and reason, consecutive-failure count.
4. Dependencies: every path through S1–S7.
5. Preconditions: none.
6. Action / side effects: run record; on any failure or degraded run, a template alert with stage, reason, providers tried, next run time and a suggested action.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries each.
9. Idempotency: record key `run_id`; alert key (`job_id`, date, stage).
10. Postconditions: run record present; alert delivered when required.
11. Verification: read back the record; alert receipt.
12. Failure routes: chat alert fails → secondary channel; both fail → record `failed_unreported` for the watcher.
13. Escalation triggers: `consecutive_failures ≥ 2` → alert states the count and the concrete fix (for example "top up provider A credit").
14. Evidence: record ID, alert message ID.

**S10 — Independent missed-run watch** (separate job at `send_time` + 30 minutes)
1. Objective: catch runs that never started or never finished.
2. Capability: `nova.ops.jobs.watch.default`; `nova.notify.owner.send.template-alert`.
3. Input mapping: expected run for today; latest run record.
4. Dependencies: none on the brief job or the model providers.
5. Preconditions: watcher job installed with the brief job.
6. Action / side effects: if no record, or a record in `failed_unreported`, send a template alert (`stage = missed_run`).
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: alert key (`job_id`, date, `missed_run`).
10. Postconditions: the owner knows by `send_time` + 30 minutes whether today's brief failed.
11. Verification: alert receipt.
12. Failure routes: channels down → retry at the next watcher tick (every 30 minutes until the end of the quiet-hour window).
13. Escalation triggers: the scheduler itself missed the run → included in the alert.
14. Evidence: watcher run record.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Fallback independence | fallback provider and account differ from primary | config + S1 | true | installation refused / `blocked` |
| Provider readiness | health known for both providers each run | S1 records | present | treat unknown as a risk; continue with disclosure |
| Feed freshness | feed age ≤ 36 h | S2 | true | disclose or alert |
| Link provenance | every link equals a selected item URL | S6 | 100 % | drop the item |
| Line grounding | names and numbers present in title, snippet or page | S6 | 100 % | headline only |
| Disclosure | `model_used` recorded; fallback footer present when used | S6, run record | true | add before sending |
| No silence | each scheduled day has a delivered brief or alert | run registry + watcher | true | watcher alert |
| No duplicates | at most one brief per day | brief key | true | suppress the duplicate |
| Output schema | `brief_result` valid | validator | valid | fix fields; never fake values |

"Job ran" is not success; only a delivery receipt for the brief or for an alert closes the day.

## J. Self-correction and recovery

- **Failure classes:** config violation, provider authentication or credit failure, rate limit, timeout, empty completion, stale feed, unreadable pages, ungrounded line, delivery failure, scheduler miss.
- **Permitted corrections:** retry the primary once for transient errors; use the declared fallback once for provider-side failures; fall back to headlines-only; use a feed up to 36 h old with disclosure; replace an ungrounded line with its headline; switch failure alerts to the secondary channel.
- **Forbidden corrections:** an undeclared model or provider; hiding a fallback; skipping the failure alert; inventing items or lines; sending to anyone but the owner.
- **Maximum correction cycles:** 1 per step; 3 per run.
- **Reverification required after correction:** yes — S6 re-runs on any changed brief.
- **Stop conditions:** both channels down (watcher takes over); correction budget exhausted.
- **Recovery or compensation:** a correction message if a wrong brief was delivered.
- **What must be reported:** fallback use, degraded mode, stale feed, headline-only items, every failure with its stage and fix.

## K. Independent challenge

- **Assumptions to challenge:** the fallback really is independent (separate provider and account, not a second model on the same router); "credit ok" from a provider is current; the top-scored items are news rather than evergreen posts.
- **Alternative approach:** if the primary has needed the fallback for several days, propose promoting the fallback or fixing the primary — owner decision.
- **Contradictory evidence:** the watcher's view versus the job's own record (a job that "succeeded" but has no delivery receipt).
- **Could mislead:** a fluent line that overstates a headline; a brief that looks fresh but uses a 36-hour-old feed.
- **Separate verifier:** S6 rules and the S10 watcher run independently of the writer and of both model providers.
- **Findings requiring disclosure:** fallback or degraded mode, feed age above 24 h, fewer than 3 items.

## L. Human escalation

- **Triggers:** 2 or more consecutive failed runs; provider credit `low` or `exhausted`; fallback used two days running; both channels failing; fallback rule violated by a config change.
- **Why the agent cannot decide:** fixing credit, keys or providers needs the owner's account access and spending decisions.
- **Information presented:** stage, error class per provider, consecutive-failure count, last good run, next scheduled run, suggested fix.
- **Decision requested:** top up or renew a provider; change providers; pause the brief.
- **Options and consequences:** pausing stops the brief and its alerts (the watcher reports "paused" once); changing providers requires the fallback rule again.
- **Safe behaviour while waiting:** the job keeps running daily; each failure is reported once per day, never silently skipped.
- **If no response:** alerts continue at most once per day per stage.
- **Resume:** after a provider change, S1 re-validates the fallback rule before the next run.

## M. Completion conditions

Mandatory: exactly one of brief / degraded brief plus alert / failure alert delivered for the day; `brief_result` valid; every link from the topic store; `model_used` and fallback disclosed; run record written; limitations disclosed; evidence references present.

- `completed` — a verified brief delivered on time (primary or disclosed fallback).
- `partially_completed` — a degraded headlines-only brief, a stale feed, fewer than 3 items, or a late delivery, each disclosed with an alert where applicable.
- `blocked` — the fallback rule or configuration invalid, or a permission missing; the owner alerted.
- `failed` — no brief could be produced or delivered; the failure alert delivered (or handed to the watcher).
- `cancelled` — the owner paused the job or stopped the run.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome ("brief delivered by fallback — primary credit exhausted"); `brief_result` and any `failure_alert`; step statuses S1–S10; verification results and failed checks; evidence references (message IDs, topic IDs, provider health records, run record ID); approvals (schedule, provider declarations); side effects (messages sent, topic statuses) and reconciliation status; corrections performed; unresolved challenges; limitations; escalation state and consecutive-failure count; learning candidate. Never include keys, account identifiers or page text.

## O. Learning and revision

- **Outcome metrics:** on-time delivery rate, fallback rate per provider, degraded-mode rate, failure alerts per stage, watcher catches, pick rate, consecutive-failure streaks.
- **Evidence retained:** run records and alert records, per the retention policy.
- **Data minimisation:** no page text or brief text in metrics.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}` — for example, "fallback used on 5 of 7 days → propose swapping primary and fallback".
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests, especially the provider-outage set.
- **Promotion approval:** publisher review plus Skill Port review; provider changes need owner approval.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (sandbox providers and channel) | 5-item brief delivered by the primary | UNVERIFIED on Skill Port; daily digest observed in the reference implementation |
| Primary out of credit | fallback writes; footer discloses it | TBD |
| Both providers down | headlines-only brief plus failure alert | TBD |
| Multi-day provider outage replay | an alert every day; escalation from day 2; never silent | TBD |
| Fallback on the same provider or account at installation | installation refused | TBD |
| Missing config (no fallback, no channel) | `blocked` at installation | TBD |
| Missing capability (fetcher absent) | lines become headline only; disclosed | TBD |
| Permission denied (`llm:invoke:fallback` missing) | installation refused; fallback cannot be activated | TBD |
| Unauthorized scope (another owner's topics) | denied | TBD |
| Tenant isolation | brief uses only the owner's store | TBD |
| Timeout and retry on the primary | 1 retry, then fallback | TBD |
| Duplicate / replay (scheduler fires twice) | one brief per day | TBD |
| False success (job "succeeded", no receipt) | watcher alert | TBD |
| Ungrounded line (adds a number) | replaced by headline only | TBD |
| Chat channel down | alert via secondary channel | TBD |
| Scheduler missed the run | watcher alert at +30 minutes | TBD |
| Correction budget exhausted | `failed` with alert | TBD |
| Escalation (credit low in pre-flight) | warning in brief and owner alert | TBD |
| Cancellation (owner pauses) | no brief; one "paused" note from the watcher | TBD |
| Compensation (wrong brief delivered) | correction message sent; reason stated | TBD |
| Output schema and status | valid `brief_result`; correct status | TBD |
| Compatibility / regression (provider API change) | detected as provider failure; fallback path exercised | TBD |

All tests use sandbox model accounts, a test chat and recorded topic fixtures. Not production-ready until the mandatory tests — especially the outage set — have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, scheduler interface, and whether the platform can run an independent watcher job — TBD (owner: publisher, with Skill Port).
- Not every provider exposes a credit or quota balance; where it does not, `credit_status = unknown` and only a real call proves readiness (owner: operator).
- The cross-provider fallback and failure reporting have no runtime evidence yet (owner: publisher).
- Per-run and monthly token caps, run-record retention — TBD (owner: operator).
- Delivery time depends on the scheduler's precision and the channel's availability.
