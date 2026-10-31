# External project watch

## A. Identity and ownership

- **Recipe ID:** `nova.ops.external-project-watch` (proposed until assigned by Skill Port)
- **Name:** External project watch
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** operations
- **Tags:** monitoring, push-model, webhook-ingest, hmac, dashboard, red-alerts, stale-feed-detection
- **Summary:** Another system the user runs (an "external project") pushes a read-only status rollup on a schedule to an authenticated ingest webhook. The agent stores each snapshot, shows the latest on a dashboard with its age, sends the owner red alerts when the numbers turn bad, and raises one alert when the feed goes stale. Nova stores no credential that could reach into the external system.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** A sender inside the external project (a read-only aggregate function plus a scheduler able to make an HTTPS POST); an HTTPS ingest endpoint on the automation hub; a snapshot store; a scheduler for the freshness check; an owner notification channel; a dashboard. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — the push → store → compare → alert → dashboard loop runs in Nova's reference implementation (a database scheduler inside the external project pushes an aggregate rollup every 3 hours; red alerts and a live dashboard panel with "as of N minutes ago"). Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.watch.feed` | Capability | Register a push feed and manage its ingest secret | medium | UNVERIFIED |
| `nova.watch.feed.register` | Operation (create) | Create a feed: ID, expected cadence, alert rules, ingest secret | medium | UNVERIFIED |
| `nova.watch.feed.rotate_secret` | Operation (update) | Issue a new ingest secret with a bounded overlap window | medium | UNVERIFIED |
| `nova.watch.ingest` | Capability | Accept authenticated rollup pushes | medium | UNVERIFIED |
| `nova.watch.ingest.receive` | Operation (read request, then hand off) | Authenticate, size-check and schema-validate one push | medium | UNVERIFIED |
| `nova.watch.snapshots` | Capability | Store and read snapshots | low | UNVERIFIED |
| `nova.watch.snapshots.append` | Operation (create) | Append one validated snapshot (append-only) | low | UNVERIFIED |
| `nova.watch.snapshots.previous` | Operation (read) | Return the comparison base (latest in-order snapshot) | low | UNVERIFIED |
| `nova.watch.snapshots.latest` | Operation (read) | Return the latest snapshot with `age_min` for the dashboard | low | UNVERIFIED |
| `nova.watch.rules` | Capability | Decide red alerts from previous vs new | low | UNVERIFIED |
| `nova.watch.rules.evaluate` | Operation (read, pure function) | Return firing, worsening and resolved rules | low | UNVERIFIED |
| `nova.watch.freshness` | Capability | Detect a stale feed | low | UNVERIFIED |
| `nova.watch.freshness.check` | Operation (read + update feed state) | Mark a feed stale or fresh, once per change | low | UNVERIFIED |
| `nova.notify.owner` | Capability | Deliver a message to the owner's own channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Send a red alert, stale alert or recovery | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.watch.ingest.receive.hmac` — **default.** The sender signs `timestamp + "." + body` with HMAC-SHA256 using the feed secret and sends the signature and timestamp in headers. The receiver recomputes, compares in constant time, and rejects timestamps outside ±5 minutes; `snapshot_id` is the replay nonce.
- `nova.watch.ingest.receive.shared-header` — opt-in, for senders that cannot compute an HMAC. A high-entropy shared secret in a request header over HTTPS. The receiver stores only a hash of the secret and compares in constant time. Replay protection relies on `snapshot_id` uniqueness and the `captured_at` ordering rule. It is weaker than `hmac`; the operator must choose it explicitly at registration and the dashboard labels the feed accordingly.
  A feed is configured with exactly one variant. There is **no fallback between them** and never a fallback to an unauthenticated mode; a push that fails its configured variant is rejected.
- `nova.watch.feed.register.default`, `nova.watch.feed.rotate_secret.default`, `nova.watch.snapshots.*.default`, `nova.watch.rules.evaluate.default`, `nova.watch.freshness.check.default`, `nova.notify.owner.send.default` — only the default variant is supported.

**Dependency outside Nova (owned by the user, UNVERIFIED):** `external.rollup-sender` — inside the external project: (a) a read-only aggregate function running with least privilege that returns only counts and integration names matching `rollup@0.1.0`, and (b) a scheduler (for example the project database's cron extension with an HTTP client, or a CI scheduled job) that POSTs the rollup to the ingest endpoint. The ingest secret is kept in the external project's own secret store and read at send time — never written into the scheduler's command text or source code. These objects live in the external project, not in Nova's backups, so the user's runbook must list them with their disable and removal steps.

**Bindings:** `binding://nova/watch/feed-register`, `binding://nova/watch/feed-rotate`, `binding://nova/watch/ingest`, `binding://nova/watch/snapshot-append`, `binding://nova/watch/snapshot-previous`, `binding://nova/watch/snapshot-latest`, `binding://nova/watch/rules`, `binding://nova/watch/freshness`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: an authenticated webhook on the automation hub that writes through a database writer function and compares with the previous snapshot; a dashboard panel that reads through a separate reader function.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Feed | `schema://nova/external-project-watch/feed@0.1.0` | the owner's tenant | create, read, update (approval) |
| Snapshot | `schema://nova/external-project-watch/snapshot@0.1.0` | one feed | create (append-only), read; delete by retention sweep only |
| Feed state | `schema://nova/external-project-watch/feed-state@0.1.0` | one feed | read, update |
| Alert | `schema://nova/external-project-watch/alert@0.1.0` | owner channel | create |
| Dashboard panel | reads `snapshot-view@0.1.0` | the owner's dashboard session | read |

**Authorization requirements (declared, never a grant):** `watch:feed:admin` (register, rotate — owner approval), `watch:ingest` (bound to one feed's secret, nothing else), `watch:snapshots:write` (ingest path only), `watch:snapshots:read` (dashboard, rules), `notify:owner`. Secrets: `secret_ref://watch-feed-<feed_id>` (HMAC key, encrypted at rest), or its hash for the shared-header variant; `secret_ref://owner-channel`. **No credential for the external project is requested or stored.**

**Policy / trust:** risk class `medium` (an internet-facing endpoint that accepts data). Pushed data is untrusted input: it is schema-validated and size-limited, its strings are never executed or rendered as HTML, and it cannot change rules, recipients or configuration.

## C. Intent and applicability

- **User problem:** The user runs another product or platform and wants to know quickly when it is unhealthy (errors, failed jobs, critical bugs, support load, integrations going idle) without giving the assistant a login, token or network path into it.
- **Desired outcome:** A dashboard panel that always says how fresh its numbers are, red alerts on meaningful changes, and an alert when the feed itself stops arriving.
- **Trigger or entry conditions:** (1) one-time feed registration by the owner; (2) each authenticated push; (3) the scheduled freshness check (default every 15 minutes); (4) the owner opening the dashboard.
- **Applicable situations:** Systems the user owns or operates and can install a small sender in: a SaaS backend, a client portal, a data platform.
- **Out-of-scope situations:** Pulling data from the external project; storing per-customer or per-person records; acting on the external project (restarting, deploying, closing tickets); monitoring systems the user does not control; public status pages.
- **Required inputs:** feed registration (see Section D) and pushes matching `rollup@0.1.0`.
- **Prerequisites:** HTTPS ingest endpoint; the user installs `external.rollup-sender`; owner channel; dashboard.
- **Required capabilities:** `nova.watch.feed`, `nova.watch.ingest`, `nova.watch.snapshots`, `nova.watch.rules`, `nova.watch.freshness`, `nova.notify.owner` (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port; the sender is a user-side dependency.
- **Assumptions:** Verified in the reference implementation: scheduled aggregate push, previous-vs-new comparison, red alerts, a dashboard with the snapshot's age. Assumed: the external project can make outbound HTTPS calls and keep a secret in its own secret store; new feeds use the signed (HMAC) variant.
- **Ambiguity policy:** Stop on anything that touches security (unknown feed, bad signature, invalid schema): reject and record. For alert thresholds, use the documented defaults.
- **Vedaxi task type:** Not applicable — this recipe does not map to any of the seven task types Vedaxi supports today.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `feed.label` | string 2–60 chars (e.g. "Client portal") | yes | owner; low | — | ask |
| `feed.cadence_minutes` | integer 15–1440 | no | owner | 180 | out of range → clamp |
| `feed.auth_variant` | enum `hmac`, `shared-header` | no | owner | `hmac` | `shared-header` needs an explicit confirmation |
| `feed.rules` | object `rules@0.1.0` | no | owner | all rules on | invalid → defaults, disclosed |
| push body | `rollup@0.1.0`, ≤ 32 KB | yes (per push) | sender; aggregate business metrics | — | invalid → 422, not stored; too large → 413; bad auth → 401, not stored |

```json
{"$id":"schema://nova/external-project-watch/feed@0.1.0","type":"object","required":["label"],"additionalProperties":false,
 "properties":{"feed_id":{"type":"string","pattern":"^feed_[a-z0-9]{8,32}$"},"label":{"type":"string","minLength":2,"maxLength":60},"cadence_minutes":{"type":"integer","minimum":15,"maximum":1440,"default":180},"auth_variant":{"enum":["hmac","shared-header"],"default":"hmac"},"rules":{"$ref":"schema://nova/external-project-watch/rules@0.1.0"},"retention_days":{"type":"integer","minimum":30,"maximum":730,"default":180},"enabled":{"type":"boolean","default":true}}}
```

```json
{"$id":"schema://nova/external-project-watch/rollup@0.1.0","type":"object","additionalProperties":false,
 "required":["feed_id","snapshot_id","captured_at","schema_version","metrics"],
 "properties":{"feed_id":{"type":"string"},"snapshot_id":{"type":"string","format":"uuid"},"captured_at":{"type":"string","format":"date-time"},
 "schema_version":{"const":"1"},"metrics":{"type":"object","additionalProperties":false,"properties":{
  "errors_24h":{"type":"integer","minimum":0},"failed_jobs_24h":{"type":"integer","minimum":0},"support_open":{"type":"integer","minimum":0},"sites_live":{"type":"integer","minimum":0},"customers":{"type":"integer","minimum":0},"users":{"type":"integer","minimum":0},
  "qa":{"type":"object","additionalProperties":false,"properties":{"open":{"type":"integer","minimum":0},"critical":{"type":"integer","minimum":0},"high":{"type":"integer","minimum":0}}},
  "integrations":{"type":"object","additionalProperties":false,"properties":{"live":{"type":"array","maxItems":50,"items":{"type":"string","pattern":"^[A-Za-z0-9 ._-]{1,40}$"}},"idle":{"type":"array","maxItems":50,"items":{"type":"string","pattern":"^[A-Za-z0-9 ._-]{1,40}$"}}}},
  "revenue":{"type":"object","additionalProperties":false,"properties":{"mrr":{"type":"number","minimum":0},"currency":{"type":"string","pattern":"^[A-Z]{3}$"}}}}}}}
```

`additionalProperties:false` at every level means a push cannot carry names, emails or free text — only counts and integration names.

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| ingest response | `ingest-response@0.1.0` | `accepted`, `snapshot_id` | the sender | low (no detail on rejection) | HTTP status matches outcome |
| alerts | `alert@0.1.0` | `feed_id`, `rule`, `kind`, `text`, `idempotency_key` | owner channel only | business metrics; owner only | delivery receipt |
| dashboard view | `snapshot-view@0.1.0` | `feed_id`, `age_min`, `freshness`, `metrics` | owner dashboard | business metrics; owner session only | age computed server-side |
| run record | `watch-run@0.1.0` | `run_id`, `status`, `path` | run log | low | Section I |

```json
{"$id":"schema://nova/external-project-watch/watch-run@0.1.0","type":"object","required":["run_id","status","path"],
 "properties":{"run_id":{"type":"string"},"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},"path":{"enum":["register","rotate","ingest","freshness","dashboard"]},"feed_id":{"type":"string"},"snapshot_id":{"type":["string","null"]},"http_status":{"type":["integer","null"]},"baseline":{"type":"boolean"},"alerts":{"type":"array","items":{"$ref":"schema://nova/external-project-watch/alert@0.1.0"}},"freshness":{"enum":["fresh","stale","unknown"]},"warnings":{"type":"array","items":{"type":"string"}}}}
```

**Other schemas** (compact JSON Schema):

```json
{"$id":"schema://nova/external-project-watch/rules@0.1.0","type":"object","properties":{"errors_level":{"type":"boolean","default":true},"failed_jobs_level":{"type":"boolean","default":true},"critical_increase":{"type":"boolean","default":true},"support_increase":{"type":"boolean","default":true},"bugs_increase":{"type":"boolean","default":true},"integration_idle":{"type":"boolean","default":true},"stale_after_cadences":{"type":"number","minimum":1.5,"maximum":6,"default":2},"stale_grace_minutes":{"type":"integer","minimum":0,"maximum":120,"default":15}}}
{"$id":"schema://nova/external-project-watch/snapshot@0.1.0","type":"object","required":["snapshot_id","feed_id","captured_at","received_at","in_order","data"],"properties":{"snapshot_id":{"type":"string"},"feed_id":{"type":"string"},"captured_at":{"type":"string","format":"date-time"},"received_at":{"type":"string","format":"date-time"},"in_order":{"type":"boolean"},"data":{"$ref":"schema://nova/external-project-watch/rollup@0.1.0"}}}
{"$id":"schema://nova/external-project-watch/snapshot-query@0.1.0","type":"object","required":["feed_id"],"properties":{"feed_id":{"type":"string"},"before":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/external-project-watch/snapshot-view@0.1.0","type":"object","required":["feed_id","age_min","freshness","metrics"],"properties":{"feed_id":{"type":"string"},"label":{"type":"string"},"captured_at":{"type":"string","format":"date-time"},"age_min":{"type":"integer","minimum":0},"freshness":{"enum":["fresh","stale","unknown"]},"auth_variant":{"enum":["hmac","shared-header"]},"metrics":{"type":"object"},"red_rules":{"type":"array","items":{"type":"string"}}}}
{"$id":"schema://nova/external-project-watch/feed-state@0.1.0","type":"object","required":["feed_id","version"],"properties":{"feed_id":{"type":"string"},"version":{"type":"integer","minimum":1},"last_snapshot_at":{"type":["string","null"],"format":"date-time"},"stale":{"type":"boolean"},"stale_alerted":{"type":"boolean"},"red":{"type":"object","additionalProperties":{"type":"object","properties":{"since":{"type":"string","format":"date-time"},"value":{"type":"number"},"last_alert_at":{"type":"string","format":"date-time"}}}},"invalid_streak":{"type":"integer","minimum":0}}}
{"$id":"schema://nova/external-project-watch/rule-result@0.1.0","type":"object","required":["firing","resolved","baseline"],"properties":{"baseline":{"type":"boolean"},"firing":{"type":"array","items":{"type":"object","required":["rule","kind"],"properties":{"rule":{"type":"string"},"kind":{"enum":["entered","worsened"]},"prev":{"type":["number","null"]},"curr":{"type":["number","string"]}}}},"resolved":{"type":"array","items":{"type":"string"}},"suspicious":{"type":"array","items":{"type":"string"}}}}
{"$id":"schema://nova/external-project-watch/alert@0.1.0","type":"object","required":["feed_id","rule","kind","text","idempotency_key"],"properties":{"feed_id":{"type":"string"},"rule":{"type":"string"},"kind":{"enum":["entered","worsened","resolved","stale","fresh_again","invalid_feed"]},"text":{"type":"string","maxLength":600},"idempotency_key":{"type":"string"}}}
{"$id":"schema://nova/external-project-watch/delivery@0.1.0","type":"object","required":["idempotency_key","delivered"],"properties":{"idempotency_key":{"type":"string"},"delivered":{"type":"boolean"},"message_id":{"type":["string","null"]}}}
{"$id":"schema://nova/external-project-watch/ingest-response@0.1.0","type":"object","required":["accepted"],"properties":{"accepted":{"type":"boolean"},"snapshot_id":{"type":["string","null"]},"duplicate":{"type":"boolean"}}}
{"$id":"schema://nova/external-project-watch/secret-issue@0.1.0","type":"object","required":["feed_id","issued_at","shown_once"],"properties":{"feed_id":{"type":"string"},"issued_at":{"type":"string","format":"date-time"},"overlap_until":{"type":["string","null"],"format":"date-time"},"shown_once":{"const":true}}}
```

Notes: `secret-issue@0.1.0`: the secret value itself is displayed to the owner once and never returned again or logged.

**Example (fictional)**

Push: `{"feed_id":"feed_k3x9q2mr","snapshot_id":"6f1c2b9e-3d4a-4e7b-9c1d-2a5b8e0f4c11","captured_at":"2026-10-31T09:00:00Z","schema_version":"1","metrics":{"errors_24h":3,"failed_jobs_24h":0,"qa":{"open":41,"critical":6,"high":20},"support_open":2,"sites_live":47,"customers":5,"users":9,"integrations":{"live":["Payments","Email"],"idle":["Search"]}}}`

Previous snapshot (06:00): `errors_24h` 0, `qa.critical` 5, `support_open` 2, `Search` was live.

Response: `202 {"accepted":true,"snapshot_id":"6f1c2b9e-…"}`. Alerts: "Client portal — RED: errors in last 24 h went 0 → 3; critical QA issues 5 → 6; integration Search moved live → idle. As of 09:00 UTC."

## E. Constraints and budgets

- **Hard constraints:** Nova never calls into the external project and stores no credential for it; every push is authenticated with the feed's configured variant before anything is stored; unauthenticated, out-of-window or replayed pushes are rejected and never stored; payloads are aggregate-only (`additionalProperties:false`) and ≤ 32 KB; the first snapshot is a baseline (no alerts); an out-of-order snapshot never becomes "latest" and never alerts; stale data is never shown as healthy; alerts go to the owner only; ingest secrets are never logged, echoed or returned after issue.
- **Soft constraints:** one message per push listing all firing rules; recoveries batched the same way.
- **Resource and tenant scope:** the owner's feeds, snapshots and feed state.
- **Data sources allowed:** authenticated pushes only.
- **External transfers permitted:** alert text to the owner channel provider; nothing is sent to the external project except the HTTP response.
- **Time limit:** ingest request ≤ 5 s end to end (alerts are delivered asynchronously after the response); freshness check ≤ 30 s.
- **Cost or token budget:** no LLM is used (alerts are deterministic, so numbers are exact).
- **Maximum steps:** 11 (S1–S11), split across paths.
- **Maximum retries and correction cycles:** the sender may retry with the same `snapshot_id` (idempotent); alert delivery 2 retries; correction cycles 1.
- **Rate limits and concurrency:** at most 12 pushes per hour per feed (429 beyond that); pushes for one feed are processed serially.
- **Retention and logging constraints:** snapshots kept `retention_days` (default 180); request logs keep status code, feed ID and size — never headers, signatures or bodies.
- **Prohibited actions:** pulling from the external project; accepting per-person data; letting pushed data change rules or recipients; rendering pushed strings as HTML.
- **Stop conditions:** feed disabled (410 to sender); authentication failure; schema failure; rate limit.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Feed (with ingest secret) | owner request | Approval Required (S1) | feed readable; secret shown once |
| Create | Snapshot (append-only) | push authenticated and valid; `snapshot_id` new | Allowed | read-back by `snapshot_id` |
| Create | Alert | rule fired or freshness changed | Allowed (owner only; scope approved at S1) | delivery receipt |
| Read | Snapshots, feed state, feed config | own tenant | Allowed | — |
| Update | Feed state: `last_snapshot_at`, `stale`, `stale_alerted`, `red`, `invalid_streak`, `version` | version compare-and-swap | Allowed | version +1 |
| Update | Feed config: cadence, rules, `enabled`; secret rotation | owner request | Approval Required | config read-back |
| Delete | Snapshots older than `retention_days` (never the latest or the comparison base) | retention sweep declared at S1 | Allowed within the declared sweep | count removed recorded |
| Delete | Feed | owner request | Approval Required | ingest returns 410 |
| External side effect | the external project | — | Not Allowed | no outbound call in logs |
| External side effect | message to the owner | rule fired | Allowed | delivery receipt |

- **Idempotency:** `snapshot_id` is unique per feed — a repeat push returns `200 {"accepted":true,"duplicate":true}` and does nothing else. Alert key = sha256(`feed_id`, `rule`, `kind`, `snapshot_id`).
- **Expected side effects:** a stored snapshot; owner messages on rule changes and freshness changes.
- **Compensation:** a wrong alert is followed by one correction message; a wrongly accepted snapshot can be marked `excluded` by the owner (excluded snapshots are skipped as comparison bases).
- **Irreversible:** delivered messages.
- **Ambiguous outcomes:** if the sender times out before the response, its retry with the same `snapshot_id` resolves as a duplicate; if alert delivery is ambiguous, the key prevents a second send where the channel deduplicates.

## G. Autonomy and approval policy

- **Default autonomy level:** A2 — bounded autonomy for ingest, alerts and freshness.
- **Planning level:** A2.
- **Execution level:** A2 for the ingest, freshness and dashboard paths; A0 (owner) for feed registration, rotation, rule changes and deletion.
- **Verification level:** A2.
- **Recovery level:** A2 for retries; A0 for excluding a snapshot or re-issuing a secret.
- **Approval required for:** registering a feed; choosing the `shared-header` variant; rotating a secret; changing rules or cadence; deleting a feed.
- **Actions never permitted:** requesting or storing credentials for the external project; calling it; accepting unauthenticated pushes; widening the payload schema without a new version; alerting anyone but the owner.
- **Conditions requiring clarification:** a registration without a label; a request to accept per-customer data (refuse and explain).
- **Conditions requiring escalation:** 3 consecutive invalid or unauthenticated pushes (possible misconfiguration or probing); the feed stale; a suspicious all-zero snapshot.
- **Safe state after stop, cancellation or escalation:** nothing stored from a rejected push; the dashboard shows the last good snapshot with its true age; no messages beyond those delivered.

Approval at registration covers owner-only alerts for that one feed. It does not cover other feeds, other recipients or any action on the external project.

## H. Execution plan

**S1 — Register the feed (one-time)**
1. Objective: create a feed and issue its ingest secret safely.
2. Capability: `nova.watch.feed.register`.
3. Input mapping: owner request → `feed@0.1.0`.
4. Dependencies: none.
5. Preconditions: the owner is authenticated on the dashboard.
6. Action / side effects: create the feed; mint a ≥ 256-bit random secret; store it as `secret_ref://watch-feed-<feed_id>` (or its hash for `shared-header`); show it once with sender-kit instructions (keep it in the external project's secret store; send the rollup on the cadence).
7. Autonomy: A0 — owner approval of label, cadence, variant, rules and retention.
8. Timeout / retry: 30 s / none.
9. Idempotency: one feed per registration request ID.
10. Postconditions: feed `enabled`; `secret-issue@0.1.0` with `shown_once=true`.
11. Verification: feed read-back; the secret is not retrievable afterwards.
12. Failure routes: storage error → nothing created, `failed`.
13. Escalation triggers: `shared-header` chosen → the owner confirms the weaker variant explicitly.
14. Evidence: feed ID, approval ID, issue time (never the secret).

**S2 — Receive and authenticate a push**
1. Objective: accept only genuine pushes from the registered sender.
2. Capability: `nova.watch.ingest.receive` (the feed's configured variant).
3. Input mapping: HTTP request → headers + raw body.
4. Dependencies: S1.
5. Preconditions: feed exists and is enabled; TLS terminated by the platform.
6. Action / side effects: reject bodies > 32 KB (413) before parsing; verify HMAC over `timestamp.body` with constant-time comparison and a ±5-minute window (or the hashed header for `shared-header`); enforce the rate limit.
7. Autonomy: A2.
8. Timeout / retry: 2 s / none (the sender retries).
9. Idempotency: read-only at this point.
10. Postconditions: an authenticated raw body, or a rejection.
11. Verification: the signature matches; the timestamp is within the window.
12. Failure routes: unknown feed or bad signature → 401 with no detail, not stored, `invalid_streak + 1`; disabled feed → 410; over limit → 429.
13. Escalation triggers: `invalid_streak` reaches 3 → one `invalid_feed` alert.
14. Evidence: feed ID, status code, body size (no headers or body).

**S3 — Validate the payload**
1. Objective: accept aggregate data only.
2. Capability: `nova.watch.ingest.receive` (schema stage).
3. Input mapping: body → `rollup@0.1.0`.
4. Dependencies: S2.
5. Preconditions: authenticated.
6. Action / side effects: parse JSON; validate strictly; `feed_id` in the body must equal the authenticated feed.
7. Autonomy: A2.
8. Timeout / retry: 1 s / none.
9. Idempotency: deterministic.
10. Postconditions: a valid rollup.
11. Verification: JSON Schema passes with `additionalProperties:false`; `captured_at` not more than 5 minutes in the future.
12. Failure routes: invalid → 422 (field path only), not stored, `invalid_streak + 1`; feed mismatch → 401.
13. Escalation triggers: as in S2.
14. Evidence: validation error paths (no values).

**S4 — Deduplicate and check order**
1. Objective: no duplicates, no replays, no time travel.
2. Capability: `nova.watch.snapshots.previous`.
3. Input mapping: `feed_id`, `snapshot_id`, `captured_at`.
4. Dependencies: S3.
5. Preconditions: valid rollup.
6. Action / side effects: read-only lookups: does `snapshot_id` exist? Is `captured_at` newer than the latest in-order snapshot?
7. Autonomy: A2.
8. Timeout / retry: 2 s / 1 retry.
9. Idempotency: read-only.
10. Postconditions: `duplicate` flag; `in_order` flag; the comparison base (or none → baseline).
11. Verification: lookups succeed.
12. Failure routes: duplicate → 200 `duplicate:true`, stop; older than latest → store with `in_order=false`, no alerts.
13. Escalation triggers: none.
14. Evidence: flags and the comparison base ID.

**S5 — Append the snapshot**
1. Objective: durable, append-only storage.
2. Capability: `nova.watch.snapshots.append`.
3. Input mapping: rollup + `received_at` + `in_order` → `snapshot@0.1.0`.
4. Dependencies: S4.
5. Preconditions: not a duplicate.
6. Action / side effects: insert one row through a writer function; a unique constraint on (`feed_id`, `snapshot_id`); no update or delete grants on this path.
7. Autonomy: A2.
8. Timeout / retry: 2 s / 1 retry (safe because of the unique constraint).
9. Idempotency: unique (`feed_id`, `snapshot_id`).
10. Postconditions: the snapshot is readable; feed state `last_snapshot_at` updated when in order.
11. Verification: read-back by `snapshot_id`.
12. Failure routes: write fails → 503 so the sender retries; nothing else happens.
13. Escalation triggers: 3 consecutive write failures → operator notice.
14. Evidence: snapshot ID, `received_at`.

**S6 — Respond to the sender**
1. Objective: tell the sender the outcome, nothing more.
2. Capability: `nova.watch.ingest.receive` (response stage).
3. Input mapping: S5 result → `ingest-response@0.1.0`.
4. Dependencies: S5.
5. Preconditions: stored, or a defined rejection.
6. Action / side effects: 202 with `snapshot_id`; rejections carry no internal detail.
7. Autonomy: A2.
8. Timeout / retry: within the 5 s budget / none.
9. Idempotency: the same request yields the same response.
10. Postconditions: the response is sent; S7 runs asynchronously.
11. Verification: status code matches the stored outcome.
12. Failure routes: connection dropped → the sender's retry resolves as a duplicate.
13. Escalation triggers: none.
14. Evidence: status code.

**S7 — Evaluate alert rules**
1. Objective: decide which red alerts and recoveries are due.
2. Capability: `nova.watch.rules.evaluate`.
3. Input mapping: comparison base, new snapshot, `rules`, feed state `red` map.
4. Dependencies: S5 (in-order snapshots only).
5. Preconditions: a comparison base exists (otherwise baseline: record and stop).
6. Action / side effects: pure function. Level rules (`errors_24h > 0`, `failed_jobs_24h > 0`) fire on entering red, re-fire only if the value at least doubles (at most once per 24 h), and resolve once when back to 0. Increase rules fire when `qa.critical`, `support_open` or `qa.open` rises. The integration rule fires only when a name is in the base's `live` list **and** the new `idle` list (never on a plain set difference, which misfires on renamed integrations). An all-zero snapshot after non-zero values is marked `suspicious`: resolutions are held for one cycle.
7. Autonomy: A2.
8. Timeout / retry: 1 s / none.
9. Idempotency: deterministic.
10. Postconditions: `rule-result@0.1.0`; feed state `red` map updated (compare-and-swap).
11. Verification: an independent recomputation (Section K) matches.
12. Failure routes: mismatch → no alerts this cycle, `partially_completed`, disclose.
13. Escalation triggers: `suspicious` → note in the alert.
14. Evidence: rule decisions with base and new values.

**S8 — Deliver red alerts**
1. Objective: tell the owner, once per change.
2. Capability: `nova.notify.owner.send`.
3. Input mapping: firing and resolved rules → one `alert@0.1.0` message per push.
4. Dependencies: S7.
5. Preconditions: something fired or resolved.
6. Action / side effects: send plain text (no markup parsing of pushed strings) to the owner with the feed label, rule, before → after values and `captured_at`.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: alert key per rule and snapshot.
10. Postconditions: delivered, or pending with the reason recorded.
11. Verification: delivery receipt.
12. Failure routes: undelivered → keep in the run record; the dashboard still shows red.
13. Escalation triggers: channel failing for 3 pushes → operator notice.
14. Evidence: message IDs.

**S9 — Freshness check (scheduled)**
1. Objective: detect a feed that stopped arriving.
2. Capability: `nova.watch.freshness.check`.
3. Input mapping: feed state, `cadence_minutes`, `stale_after_cadences`, `stale_grace_minutes`, now.
4. Dependencies: S1.
5. Preconditions: feed enabled and at least one snapshot ever received (otherwise `unknown`, no alert).
6. Action / side effects: stale iff `now − last_snapshot_at > cadence × stale_after_cadences + grace` (default 6 h 15 min); on fresh→stale send one `stale` alert; on the next in-order snapshot send one `fresh_again`.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 1 retry.
9. Idempotency: `stale_alerted` flag with compare-and-swap.
10. Postconditions: feed state `stale` correct.
11. Verification: the flag matches the computed age.
12. Failure routes: check fails → the dashboard computes age itself, so staleness still shows.
13. Escalation triggers: stale for more than 24 h → included in the owner's daily brief, if that recipe is enabled.
14. Evidence: computed age, flag transitions.

**S10 — Serve the dashboard panel**
1. Objective: show the latest numbers honestly.
2. Capability: `nova.watch.snapshots.latest`.
3. Input mapping: `feed_id` → `snapshot-view@0.1.0`.
4. Dependencies: S5.
5. Preconditions: owner dashboard session.
6. Action / side effects: read-only; compute `age_min` server-side; label stale feeds and the `shared-header` variant; escape all pushed strings.
7. Autonomy: A2.
8. Timeout / retry: 3 s / 1 retry.
9. Idempotency: read-only.
10. Postconditions: a view with `freshness` and `age_min`.
11. Verification: `freshness = stale` whenever the S9 rule says so, even if S9 has not run.
12. Failure routes: no snapshot → "waiting for first push".
13. Escalation triggers: none.
14. Evidence: none beyond access logs.

**S11 — Rotate the secret (on request)**
1. Objective: replace the ingest secret without losing pushes.
2. Capability: `nova.watch.feed.rotate_secret`.
3. Input mapping: `feed_id`, overlap window (default 24 h, maximum 7 days).
4. Dependencies: S1.
5. Preconditions: owner approval.
6. Action / side effects: mint a new secret, show it once; accept old and new until `overlap_until`; then the old one is revoked.
7. Autonomy: A0 approval, then A2.
8. Timeout / retry: 30 s / none.
9. Idempotency: one rotation per request ID.
10. Postconditions: `secret-issue@0.1.0` with `overlap_until`.
11. Verification: a push signed with the new secret succeeds; after the window, the old one fails.
12. Failure routes: storage error → the old secret stays valid, `failed`.
13. Escalation triggers: suspected leak → the owner may choose zero overlap.
14. Evidence: rotation time, overlap end (never the secrets).

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Authentication | every stored snapshot passed its feed's variant | ingest log | 100 % | purge the row, investigate, alert |
| No inbound credential | no secret for the external project exists in Nova | secret inventory | none | stop, escalate |
| Payload minimality | stored data validates with `additionalProperties:false` | validator | valid | reject |
| Replay | no two snapshots share `snapshot_id` | unique constraint | holds | duplicate response |
| Order | only in-order snapshots update latest or alert | S4 flags | holds | correct the latest pointer |
| Baseline | first snapshot produces no alerts | S7 result | zero alerts | discard alerts |
| Rule accuracy | alert numbers equal snapshot values | alert text vs stored rows | exact | correction message |
| Freshness truth | dashboard `stale` whenever the age exceeds the threshold | view vs computed age | holds | fix the view |
| Secret hygiene | no secret, signature or body in any log | log scan | none | rotate the secret, purge logs |
| Output schema | responses and views valid | validator | valid | fix; never fake |

A 202 response proves only that the snapshot was stored, not that the external project is healthy.

## J. Self-correction and recovery

- **Failure classes:** authentication failure, schema failure, oversize, rate limit, duplicate, out-of-order, store failure, rule-check mismatch, delivery failure, stale feed.
- **Permitted corrections:** return a retryable status so the sender retries; treat duplicates idempotently; hold resolutions after a suspicious snapshot; recompute rules once.
- **Forbidden corrections:** accept a push that failed authentication "just this once"; loosen the schema at runtime; switch a feed's variant without approval; pull data from the external project to fill a gap; show stale data as fresh.
- **Maximum correction cycles:** 1 per push.
- **Reverification required after correction:** yes — re-run S3 validation or the S7 recomputation.
- **Stop conditions:** hard constraint fails; feed disabled.
- **Recovery or compensation procedure:** exclude a wrong snapshot (owner); send a correction message; rotate a suspected-leaked secret.
- **What must be reported:** rejections by class, held resolutions, stale periods, rotations.

## K. Independent challenge

- **Assumptions to challenge:** the sender's aggregate definitions have not changed (a jump may be a query change, not reality); the sender's clock is right; a quiet feed is a broken sender, not a quiet project.
- **Alternative explanation:** an all-zero snapshot is more likely a failed query than a perfect day — held as `suspicious`.
- **Contradictory evidence:** a "resolved" errors rule while failed jobs rise; integrations simultaneously live and idle.
- **Could mislead:** a dashboard showing old numbers without their age; a renamed integration read as "went idle".
- **Separate verifier:** a second rule implementation (plain comparisons, no shared code) recomputes S7 from the two stored rows; mismatches block alerts for that cycle.
- **Findings requiring disclosure:** suspicious snapshots, schema-version changes, out-of-order arrivals.

## L. Human escalation

- **Triggers:** 3 consecutive invalid or unauthenticated pushes; the feed stale; a suspicious snapshot; a suspected secret leak.
- **Why the agent cannot decide:** fixing the sender, rotating secrets and judging business severity are owner decisions, and the agent has (by design) no access to the external project.
- **Information presented:** feed label, failure class and count, last good snapshot time, current dashboard state.
- **Decision requested:** fix the sender; rotate the secret; exclude a snapshot; pause the feed.
- **Options and consequences:** pausing returns 410 to the sender and stops alerts; rotating with zero overlap breaks the sender until it is updated.
- **Safe behaviour while waiting:** keep rejecting bad pushes; keep showing the true age.
- **If no response:** nothing changes; the stale alert is not repeated.
- **Resume:** the next valid push clears the condition and sends one `fresh_again`.

## M. Completion conditions

Mandatory: every stored snapshot authenticated and schema-valid; no credential for the external project held; baseline respected; alerts match stored values; freshness truthful; secrets never logged; evidence references present; limitations disclosed.

- `completed` — (ingest) stored, rules evaluated, due alerts delivered; (freshness) state correct; (register/rotate) secret issued once.
- `partially_completed` — stored but alert delivery failed or the rule recomputation disagreed.
- `blocked` — rejected for authentication, schema, size, rate limit or a disabled feed.
- `failed` — the store failed (sender told to retry) or registration failed.
- `cancelled` — the owner declined registration or rotation.

A rejected push is `blocked`, never `completed`, even though the endpoint responded correctly.

## N. Final output contract

Return per run: run ID; recipe ID and version; path; status; one-line outcome; `watch-run@0.1.0`; step statuses for the path; verification results; evidence references (snapshot ID, alert keys, message IDs); approvals and scope (registration or rotation approval); side effects (snapshot stored, messages sent) and reconciliation status; corrections; unresolved challenges (suspicious snapshot); limitations; escalation state; learning candidate. Never include secrets, signatures, request headers or raw bodies.

## O. Learning and revision

- **Outcome metrics:** pushes accepted vs rejected by class, median push latency, stale periods per month, alerts per week, alerts the owner marked "not useful".
- **Evidence retained:** run records and snapshot IDs, per retention policy.
- **Data minimisation:** metrics hold counts and feed IDs only.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}`.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review; a schema widening needs a new `schema_version` and owner approval per feed.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (valid signed push, errors 0 → 3) | 202, stored, one red alert | UNVERIFIED on Skill Port; observed in the reference implementation |
| First snapshot | baseline, no alerts | UNVERIFIED on Skill Port; observed in the reference implementation |
| Green → green | stored, no alert | TBD |
| Missing input (registration without label) | asks; nothing created | TBD |
| Invalid input (extra field `customer_email`) | 422, not stored | TBD |
| Oversize body | 413 before parsing | TBD |
| Missing capability (snapshot store absent) | 503; `blocked` for activation | TBD |
| Permission denied (no signature) | 401, not stored | TBD |
| Unauthorized scope (feed A's secret signs feed B's body) | 401 | TBD |
| Tenant isolation | cannot read another tenant's snapshots | TBD |
| Timestamp outside window | 401 | TBD |
| Duplicate / replay (same `snapshot_id`) | 200 `duplicate:true`, no second alert | TBD |
| Out-of-order snapshot | stored `in_order=false`, no alert | TBD |
| Timeout on store | 503; retry stores once | TBD |
| False success (all-zero snapshot after red) | `suspicious`; resolution held one cycle | TBD |
| Integration renamed | no false "went idle" alert | TBD |
| Failed verification (recomputation mismatch) | alerts held, `partially_completed` | TBD |
| Stale feed | one `stale` alert; one `fresh_again` | TBD |
| Correction budget exhausted | stops safely | TBD |
| Escalation (3 bad pushes) | one `invalid_feed` alert | TBD |
| Cancellation (owner declines registration) | `cancelled`, nothing created | TBD |
| Compensation (wrong snapshot) | excluded; correction message | TBD |
| Secret rotation overlap | old and new both accepted until overlap end | TBD |
| Log hygiene | no secret, signature or body in logs | TBD |
| Output schema and status | valid response, view and run record | TBD |
| Compatibility / regression (`schema_version` "2") | rejected until approved | TBD |

All tests use a test feed and fictional metrics; never a production sender. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, schema registry and the approval UI — TBD (owner: publisher, with Skill Port).
- Whether the platform's webhook gateway can verify HMAC signatures natively or needs an adapter — UNVERIFIED.
- Per-client or per-project breakdowns are out of scope (aggregate-only by design); adding them needs a new schema version and a privacy review.
- The sender kit lives in the user's system; Nova cannot verify its least-privilege setup — the user must review it (owner: user).
- The all-zero "suspicious" heuristic and the default stale threshold (2 cadences + 15 minutes) are design defaults, not measured values.
- Snapshot retention default (180 days) — operator to confirm.
