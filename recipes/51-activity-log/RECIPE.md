# Activity log

## A. Identity and ownership

- **Recipe ID:** `nova.ops.activity-log` (proposed until assigned by Skill Port)
- **Name:** Activity log
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** operations
- **Tags:** audit-trail, append-only, database-triggers, logging-gateway, redaction, search, tamper-evidence
- **Summary:** One append-only record of everything the assistant does. Meetings and calls are logged automatically by database triggers in the same transaction as the change; other actions (emails sent, messages, media, custom actions) are logged through an authenticated logging gateway. The owner can search and filter the log, and entries never contain secrets or raw payloads.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** A PostgreSQL-compatible database with triggers and security-definer functions; an HTTPS logging gateway on the automation hub; an owner dashboard for search. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — the activity table, the logging function, triggers for meetings and calls (and for minutes, research briefs and published posts), the logging gateway, and a searchable feed with source filter, failed-only toggle, pagination and day separators run in Nova's reference implementation; not every control described here has evidence yet (Section P lists the tests still needed). Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.activity.log` | Capability | Append, search and verify the audit trail | medium | UNVERIFIED |
| `nova.activity.log.append` | Operation (create) | Validate, redact and append one entry | medium | UNVERIFIED |
| `nova.activity.log.feed` | Operation (read) | Search and filter entries with cursor pagination | medium | UNVERIFIED |
| `nova.activity.log.stats` | Operation (read) | Counts today, total, by source and by status | low | UNVERIFIED |
| `nova.activity.log.verify` | Operation (read) | Verify the hash chain and coverage against domain tables | low | UNVERIFIED |
| `nova.activity.triggers` | Capability | Log domain changes automatically | medium | UNVERIFIED |
| `nova.activity.triggers.install` | Operation (create) | Install or update the trigger map on domain tables | medium | UNVERIFIED |
| `nova.activity.retention` | Capability | Enforce the retention period | medium | UNVERIFIED |
| `nova.activity.retention.purge` | Operation (delete, policy-bound) | Remove entries older than the retention period and leave a tombstone | medium | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.activity.log.append.db-trigger` — an AFTER trigger on a domain table calls the logging function in the same transaction. If the append fails, the domain change rolls back, so no action can happen unlogged. Used for meetings (booked, cancelled, rescheduled), calls (placed, received on end), minutes created, research briefs and published posts.
- `nova.activity.log.append.in-transaction` — another recipe's database function (for example the approval queue) calls the logging function inside its own transaction. Same guarantees as `db-trigger`.
- `nova.activity.log.append.gateway` — an authenticated HTTPS endpoint for actions that have no domain table (emails sent, chat messages sent, media generated, custom actions). Each writer has its own credential that may append only for its declared sources.
  Selection is by where the writer runs. There is **no fallback between variants**: if the gateway is down, a writer retries the gateway with the same `request_key`; it never switches to a direct database write, because that would bypass writer-to-source binding.
- `nova.activity.log.feed|stats|verify.default`, `nova.activity.triggers.install.default`, `nova.activity.retention.purge.default` — only the default variant is supported.

**Bindings:** `binding://nova/activity/append-fn`, `binding://nova/activity/gateway`, `binding://nova/activity/feed`, `binding://nova/activity/stats`, `binding://nova/activity/verify`, `binding://nova/activity/triggers`, `binding://nova/activity/purge` — all `status: UNVERIFIED`. Reference implementation: an activity table written only through a security-definer logging function, triggers on domain tables, a webhook gateway on the automation hub, and a dashboard page that reads through a feed function.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Activity entry | `schema://nova/activity-log/activity-entry@0.1.0` | the owner's tenant | create, read; delete only by retention purge |
| Writer registration | `schema://nova/activity-log/writer@0.1.0` | operator | read |
| Trigger map | `schema://nova/activity-log/trigger-map@0.1.0` | operator | create, update (approval) |
| Tombstone | an activity entry with `action = retention_purge` | the owner's tenant | create |
| Integrity report | `schema://nova/activity-log/integrity-report@0.1.0` | the owner's tenant | create (run output) |

**Authorization requirements (declared, never a grant):** `activity:append` (logging function and gateway writers; a writer is bound to its declared sources), `activity:read` (owner dashboard), `activity:verify` (scheduler), `activity:triggers:admin` (operator, approval), `activity:retention` (dedicated retention role, the only role with delete rights). No role has UPDATE on entries. Secrets by reference: `secret_ref://activity-writer-<writer_id>` per gateway writer.

**Policy / trust:** risk class `medium` — it records personal activity and must stay trustworthy. Incoming titles, details and metadata are untrusted text: they are length-limited, scanned for secrets, never executed, and escaped when shown.

## C. Intent and applicability

- **User problem:** The assistant books meetings, places calls, sends emails and publishes posts across many tools; the owner has no single place to see what it did, when, and whether it worked.
- **Desired outcome:** A complete, searchable, tamper-evident timeline where every action has one entry with time, actor, source, action, status and a link to the underlying record — and nothing sensitive inside.
- **Trigger or entry conditions:** (1) a domain change on a table with an installed trigger; (2) a writer posting to the gateway; (3) the owner searching the feed; (4) scheduled integrity verification and retention.
- **Applicable situations:** Any action the assistant takes on the owner's behalf; owner actions taken through the dashboard; system events (jobs failing, approvals expiring).
- **Out-of-scope situations:** Storing message bodies, transcripts, documents or request payloads; general application logging or debugging; logging other people's activity outside the owner's tenant; editing past entries.
- **Required inputs:** `activity-input@0.1.0` (see Section D).
- **Prerequisites:** Database with trigger support; logging function installed; gateway with per-writer credentials; dashboard.
- **Required capabilities:** `nova.activity.log`, `nova.activity.triggers`, `nova.activity.retention` (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port; every binding is a dependency to resolve before activation.
- **Assumptions:** Verified in the reference implementation: triggers for meetings and calls, the gateway, the searchable feed. Not every control in this recipe has evidence yet; all of them are mandatory before activation.
- **Ambiguity policy:** Use documented safe defaults: unknown `status` → `ok` is never assumed (rejected); unknown source → rejected; over-long text → truncated with a marker.
- **Vedaxi task type:** Not applicable — this recipe does not map to any of the seven task types Vedaxi supports today.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `source` | registered enum, e.g. `calendar`, `voice`, `mail`, `content`, `research`, `minutes`, `approvals`, `media`, `system` | yes | writer | — | unknown → reject |
| `action` | snake_case verb phrase, e.g. `meeting_booked`, `email_sent` | yes | writer | — | bad pattern → reject |
| `title` | string 3–140 chars | yes | writer; may name a person | — | longer → truncated with "…" |
| `detail` | string ≤ 500 chars, a human summary | no | writer | `null` | longer → truncated; payload-like content (JSON, base64) → rejected |
| `status` | enum `ok`, `pending`, `failed` | yes | writer | — | missing → reject |
| `actor` | enum `assistant`, `owner`, `system` | no | writer context | `assistant` | — |
| `entity_id` | string ≤ 64 | no | writer | `null` | — |
| `ref_path` | dashboard-relative path `^/[A-Za-z0-9/_-]{0,200}$` | no | writer | `null` | absolute URLs or query strings → dropped |
| `meta` | object, allowlisted keys per source, ≤ 2 KB | no | writer | `{}` | unknown keys dropped |
| `request_key` | string 8–128, unique per writer | yes (gateway) | writer | derived from the row for triggers | missing on gateway → reject |

```json
{"$id":"schema://nova/activity-log/activity-input@0.1.0","type":"object","additionalProperties":false,"required":["source","action","title","status"],
 "properties":{"source":{"type":"string","pattern":"^[a-z]{2,20}$"},"action":{"type":"string","pattern":"^[a-z]+(_[a-z]+){0,5}$"},
 "title":{"type":"string","minLength":3,"maxLength":140},"detail":{"type":["string","null"],"maxLength":500},
 "status":{"enum":["ok","pending","failed"]},"actor":{"enum":["assistant","owner","system"],"default":"assistant"},
 "entity_id":{"type":["string","null"],"maxLength":64},"ref_path":{"type":["string","null"],"pattern":"^/[A-Za-z0-9/_-]{0,200}$"},
 "meta":{"type":"object","maxProperties":12,"additionalProperties":{"type":["string","number","boolean","null"]}},
 "request_key":{"type":"string","minLength":8,"maxLength":128}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| activity entry | `activity-entry@0.1.0` | `id`, `ts`, `source`, `action`, `status`, `entry_hash` | activity table | personal (owner's own activity) | read-back; chain check |
| append result | `append-result@0.1.0` | `id`, `duplicate`, `redacted` | writer | low | — |
| feed page | `feed-page@0.1.0` | `items`, `next_cursor` | owner dashboard | owner session only | filters applied server-side |
| integrity report | `integrity-report@0.1.0` | `chain_ok`, `checked`, `coverage` | run record, jobs board | low | Section I |

```json
{"$id":"schema://nova/activity-log/activity-entry@0.1.0","type":"object",
 "required":["id","ts","tenant_id","writer_id","source","action","title","status","actor","prev_hash","entry_hash"],
 "properties":{"id":{"type":"integer","minimum":1},"ts":{"type":"string","format":"date-time"},"tenant_id":{"type":"string"},"writer_id":{"type":"string"},
 "source":{"type":"string"},"action":{"type":"string"},"title":{"type":"string","maxLength":140},"detail":{"type":["string","null"],"maxLength":500},
 "status":{"enum":["ok","pending","failed"]},"actor":{"enum":["assistant","owner","system"]},"entity_id":{"type":["string","null"]},
 "ref_path":{"type":["string","null"]},"meta":{"type":"object"},"redacted":{"type":"boolean"},
 "prev_hash":{"type":"string","pattern":"^[a-f0-9]{64}$"},"entry_hash":{"type":"string","pattern":"^[a-f0-9]{64}$"}}}
```

`entry_hash` = SHA-256(`prev_hash` ‖ canonical JSON of the entry without `entry_hash`); appends for one tenant are serialised so the chain has no forks.

**Other schemas** (compact JSON Schema):

```json
{"$id":"schema://nova/activity-log/append-result@0.1.0","type":"object","required":["id","duplicate","redacted"],"properties":{"id":{"type":"integer"},"duplicate":{"type":"boolean"},"redacted":{"type":"boolean"},"truncated":{"type":"boolean"}}}
{"$id":"schema://nova/activity-log/feed-query@0.1.0","type":"object","properties":{"q":{"type":"string","maxLength":100},"source":{"type":"string"},"action":{"type":"string"},"status":{"enum":["ok","pending","failed"]},"actor":{"enum":["assistant","owner","system"]},"from":{"type":"string","format":"date-time"},"to":{"type":"string","format":"date-time"},"cursor":{"type":["string","null"]},"limit":{"type":"integer","minimum":1,"maximum":100,"default":30}}}
{"$id":"schema://nova/activity-log/feed-page@0.1.0","type":"object","required":["items","next_cursor"],"properties":{"items":{"type":"array","items":{"$ref":"schema://nova/activity-log/activity-entry@0.1.0"}},"next_cursor":{"type":["string","null"]},"total_estimate":{"type":"integer"}}}
{"$id":"schema://nova/activity-log/stats@0.1.0","type":"object","required":["today","total","by_source","failed_24h"],"properties":{"today":{"type":"integer"},"total":{"type":"integer"},"by_source":{"type":"object","additionalProperties":{"type":"integer"}},"failed_24h":{"type":"integer"}}}
{"$id":"schema://nova/activity-log/integrity-report@0.1.0","type":"object","required":["chain_ok","checked","coverage","run_at"],"properties":{"chain_ok":{"type":"boolean"},"checked":{"type":"integer"},"first_bad_id":{"type":["integer","null"]},"coverage":{"type":"array","items":{"type":"object","required":["domain","expected","logged"],"properties":{"domain":{"type":"string"},"expected":{"type":"integer"},"logged":{"type":"integer"}}}},"run_at":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/activity-log/verify-request@0.1.0","type":"object","required":["since"],"properties":{"since":{"type":"string","format":"date-time"},"full":{"type":"boolean","default":false}}}
{"$id":"schema://nova/activity-log/writer@0.1.0","type":"object","required":["writer_id","sources","variant"],"properties":{"writer_id":{"type":"string"},"sources":{"type":"array","items":{"type":"string"}},"variant":{"enum":["db-trigger","in-transaction","gateway"]},"meta_keys":{"type":"object","additionalProperties":{"type":"array","items":{"type":"string"}}},"enabled":{"type":"boolean"}}}
{"$id":"schema://nova/activity-log/trigger-map@0.1.0","type":"object","required":["domain","events"],"properties":{"domain":{"type":"string"},"events":{"type":"array","items":{"type":"object","required":["on","action"],"properties":{"on":{"enum":["insert","status_change","field_set"]},"field":{"type":"string"},"value":{"type":"string"},"action":{"type":"string"},"title_template":{"type":"string","maxLength":140}}}}}}
{"$id":"schema://nova/activity-log/purge-request@0.1.0","type":"object","required":["older_than_days"],"properties":{"older_than_days":{"type":"integer","minimum":30},"dry_run":{"type":"boolean","default":true}}}
{"$id":"schema://nova/activity-log/purge-result@0.1.0","type":"object","required":["removed","tombstone_id","range"],"properties":{"removed":{"type":"integer"},"tombstone_id":{"type":["integer","null"]},"range":{"type":"object","properties":{"from_id":{"type":"integer"},"to_id":{"type":"integer"},"last_hash":{"type":"string"}}}}}
```

**Example (fictional)**

Gateway input: `{"source":"mail","action":"email_sent","title":"Recap email sent to Jane Doe","detail":"Reply in thread 'Thursday recap' (approved apr_7k2m9q4x1z8w).","status":"ok","entity_id":"msg_3391","ref_path":"/mail/msg_3391","meta":{"approval_id":"apr_7k2m9q4x1z8w"},"request_key":"mail-send-msg_3391"}`

Stored entry: `{"id":10442,"ts":"2026-11-02T15:05:02Z","tenant_id":"t_01","writer_id":"w_mail","source":"mail","action":"email_sent","title":"Recap email sent to Jane Doe","detail":"Reply in thread 'Thursday recap' (approved apr_7k2m9q4x1z8w).","status":"ok","actor":"assistant","entity_id":"msg_3391","ref_path":"/mail/msg_3391","meta":{"approval_id":"apr_7k2m9q4x1z8w"},"redacted":false,"prev_hash":"<64 hex>","entry_hash":"<64 hex>"}`

Trigger example: inserting a booking for "Intro call — Jane Doe" writes `{"source":"calendar","action":"meeting_booked","title":"Meeting booked: Intro call — Jane Doe","entity_id":"bk_208","ref_path":"/meetings/bk_208","status":"ok","actor":"system"}` in the same transaction.

## E. Constraints and budgets

- **Hard constraints:** append-only — no role can UPDATE entries, and only the retention role can DELETE, and only entries older than the approved retention period; triggers and in-transaction appends commit atomically with the domain change (no unlogged action); entries never contain secrets, credentials, tokens, message bodies, transcripts, request payloads, full email addresses or full phone numbers (masked in `title` and `detail`); gateway writers authenticate individually and may append only for their declared sources; every entry is chained by hash; the feed is readable only by the owner's session; stored text is escaped when shown.
- **Soft constraints:** titles readable at a glance ("Meeting booked: …"); one entry per real action, not per retry.
- **Resource and tenant scope:** the owner's tenant only.
- **Data sources allowed:** domain-table changes with installed triggers; registered writers.
- **External transfers permitted:** none.
- **Time limit:** append ≤ 200 ms inside a transaction, ≤ 2 s through the gateway; feed query ≤ 2 s.
- **Cost or token budget:** no LLM is used.
- **Maximum steps:** 10 (S1–S10), split across paths.
- **Maximum retries and correction cycles:** gateway writers: 3 retries with the same `request_key`; verification: 1 rerun; correction cycles: 1.
- **Rate limits and concurrency:** gateway 60 appends per minute per writer; appends serialised per tenant for the chain.
- **Retention and logging constraints:** retention period set by the operator (`TBD`; suggested 2 years); purges leave a tombstone that records the removed range and the last hash; gateway access logs keep writer ID and status only.
- **Prohibited actions:** editing or back-dating entries; deleting entries outside the retention purge; logging payloads; reading the log from automations that do not need it.
- **Stop conditions:** writer unauthenticated or not bound to the source; schema invalid; chain lock unavailable (the transaction fails and the writer retries).

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Activity entry | valid input; writer authorised for the source; redaction applied | Allowed | read-back; chain link valid |
| Create | Trigger map / triggers on a domain table | operator request | Approval Required | test row in a rolled-back transaction produces one entry |
| Create | Tombstone entry | retention purge ran | Allowed (policy) | tombstone present |
| Read | Entries, stats | owner session (feed); scheduler (verify) | Allowed | — |
| Update | any entry field | — | Not Allowed | trigger raises on UPDATE |
| Delete | Entries older than the retention period | approved retention policy; dry run reviewed once | Approval Required (policy, once) then Allowed by schedule | tombstone with range and last hash |
| External side effect | none | — | Not applicable | — |

- **Idempotency:** gateway: unique (`writer_id`, `request_key`) — a repeat returns the existing ID with `duplicate=true`. Triggers: the transaction itself is the unit; a rolled-back change leaves no entry.
- **Expected side effects:** one entry per action; a tombstone per purge.
- **Compensation:** a wrong entry is never edited; a correcting entry (`action = correction`, `meta.corrects = <id>`) is appended.
- **Irreversible:** purged entries (only their range and last hash survive in the tombstone).
- **Ambiguous outcomes:** a gateway timeout is resolved by retrying with the same `request_key`.

## G. Autonomy and approval policy

- **Default autonomy level:** A2 — bounded autonomy (appends, verification and scheduled retention within approved policy).
- **Planning level:** A2.
- **Execution level:** A2 for appends and verification; A0 for installing triggers and approving the retention policy.
- **Verification level:** A2.
- **Recovery level:** A2 for gateway retries; A0 for anything touching chain integrity.
- **Approval required for:** installing or changing triggers; registering a gateway writer and its sources; setting or changing the retention period.
- **Actions never permitted:** updating or deleting entries outside retention; logging payloads or secrets; granting `activity:append` without a source binding; disabling a trigger to make a domain write succeed.
- **Conditions requiring clarification:** a writer asks for a new source or meta key (operator decides).
- **Conditions requiring escalation:** chain verification fails; coverage gap between a domain table and its entries; repeated secret redactions from one writer.
- **Safe state after stop, cancellation or escalation:** the log stays append-only and readable; no entries are altered; a failed append rolls back its own domain change.

## H. Execution plan

**S1 — Validate the entry**
1. Objective: accept only well-formed, bounded entries.
2. Capability: `nova.activity.log.append` (validation stage, any variant).
3. Input mapping: gateway body, trigger row mapping (`trigger-map@0.1.0`) or in-transaction call → `activity-input@0.1.0`.
4. Dependencies: none.
5. Preconditions: source registered.
6. Action / side effects: schema validation; truncate over-long text with "…"; drop `ref_path` values that are not dashboard-relative; drop unknown meta keys.
7. Autonomy: A2.
8. Timeout / retry: inside the append budget / none.
9. Idempotency: deterministic.
10. Postconditions: a valid input.
11. Verification: JSON Schema passes.
12. Failure routes: invalid → gateway 422 (field paths only); trigger path → the transaction fails with a clear error and the writer's caller retries after fixing the mapping.
13. Escalation triggers: a trigger mapping failing in tests → block installation (S-install is approval-gated).
14. Evidence: validation result.

**S2 — Authenticate and authorise the writer**
1. Objective: only registered writers, only for their sources.
2. Capability: gateway authentication; database role check.
3. Input mapping: gateway credential → `writer@0.1.0`; trigger/in-transaction → the function's owning role.
4. Dependencies: S1.
5. Preconditions: writer enabled.
6. Action / side effects: verify the credential (constant-time); check `source ∈ writer.sources`; check the rate limit.
7. Autonomy: A2.
8. Timeout / retry: 1 s / none.
9. Idempotency: read-only.
10. Postconditions: an authorised writer ID.
11. Verification: binding check result.
12. Failure routes: unauthenticated → 401; source not bound → 403; over limit → 429; none are stored.
13. Escalation triggers: repeated 401/403 from one credential → operator notice.
14. Evidence: writer ID, decision.

**S3 — Redact and minimise**
1. Objective: no secrets, no payloads, minimal personal data.
2. Capability: logging function (redaction stage).
3. Input mapping: validated input.
4. Dependencies: S2.
5. Preconditions: none.
6. Action / side effects: replace matches for bearer tokens, JWT-shaped strings, private-key blocks, common API-key prefixes, `password=`-style pairs and card-number-like digit runs with `[REDACTED:<type>]`; mask email addresses (first letter + domain) and phone numbers (last two digits); reject `detail` that looks like a payload (JSON objects, long base64).
7. Autonomy: A2.
8. Timeout / retry: within the append budget / none.
9. Idempotency: deterministic.
10. Postconditions: a minimised input; `redacted` flag set when anything was replaced.
11. Verification: a second scan of the output finds no pattern match.
12. Failure routes: payload-like detail → reject (422 or transaction error).
13. Escalation triggers: more than 3 redactions from one writer in a day → operator notice (the writer is sending what it should not).
14. Evidence: redaction count by type (never the matched text).

**S4 — Deduplicate**
1. Objective: one entry per action.
2. Capability: logging function (idempotency stage).
3. Input mapping: `writer_id`, `request_key`.
4. Dependencies: S3.
5. Preconditions: gateway path (trigger paths use the transaction as the unit).
6. Action / side effects: look up (`writer_id`, `request_key`).
7. Autonomy: A2.
8. Timeout / retry: within budget / none.
9. Idempotency: unique constraint.
10. Postconditions: `duplicate` flag.
11. Verification: lookup result.
12. Failure routes: duplicate → return the existing ID; stop.
13. Escalation triggers: none.
14. Evidence: existing ID when duplicate.

**S5 — Append with the hash chain**
1. Objective: durable, chained, append-only storage.
2. Capability: `nova.activity.log.append` (variant by path).
3. Input mapping: minimised input + tenant + writer → `activity-entry@0.1.0`.
4. Dependencies: S4.
5. Preconditions: per-tenant chain lock obtainable.
6. Action / side effects: take the tenant's transaction-scoped lock; read the last `entry_hash`; compute `entry_hash`; insert. For triggers and in-transaction calls, this happens inside the caller's transaction.
7. Autonomy: A2.
8. Timeout / retry: lock wait ≤ 2 s / the gateway retries the whole request with the same key.
9. Idempotency: unique (`writer_id`, `request_key`); transaction atomicity for triggers.
10. Postconditions: one new entry linked to the previous one.
11. Verification: `prev_hash` equals the previous entry's `entry_hash`.
12. Failure routes: lock timeout or insert error → transaction fails; trigger path rolls back the domain change; gateway returns 503.
13. Escalation triggers: lock timeouts above 1 % of appends in an hour → operator notice.
14. Evidence: entry ID, `entry_hash`.

**S6 — Confirm the append**
1. Objective: the writer knows the entry exists.
2. Capability: logging function return value.
3. Input mapping: inserted row → `append-result@0.1.0`.
4. Dependencies: S5.
5. Preconditions: transaction committed (gateway) or will commit with the caller (trigger).
6. Action / side effects: return ID, `duplicate`, `redacted`, `truncated`.
7. Autonomy: A2.
8. Timeout / retry: within budget / none.
9. Idempotency: same request → same result.
10. Postconditions: result returned.
11. Verification: the ID is readable after commit.
12. Failure routes: response lost → writer retries with the same key → duplicate result.
13. Escalation triggers: none.
14. Evidence: append result.

**S7 — Search the feed**
1. Objective: let the owner find any action quickly.
2. Capability: `nova.activity.log.feed`; `nova.activity.log.stats`.
3. Input mapping: dashboard filters → `feed-query@0.1.0`.
4. Dependencies: entries exist.
5. Preconditions: owner session.
6. Action / side effects: read-only; text search over `title` and `detail`; filters by source, action, status (including "failed only"), actor and time range; keyset pagination on (`ts`, `id`); results grouped by day in the UI; all text escaped.
7. Autonomy: A2.
8. Timeout / retry: 2 s / 1 retry.
9. Idempotency: read-only.
10. Postconditions: `feed-page@0.1.0`.
11. Verification: every returned item matches the filters (server-side).
12. Failure routes: timeout → narrower default window (last 7 days) with a notice.
13. Escalation triggers: none.
14. Evidence: none beyond access logs.

**S8 — Verify integrity (scheduled)**
1. Objective: prove the log is unbroken and complete.
2. Capability: `nova.activity.log.verify`.
3. Input mapping: `verify-request@0.1.0` (daily incremental since the last verified ID; weekly full).
4. Dependencies: S5.
5. Preconditions: scheduler running.
6. Action / side effects: recompute hashes along the chain; compare domain-table event counts (for example meetings booked yesterday) with matching entries; write the report.
7. Autonomy: A2.
8. Timeout / retry: 5 minutes / 1 rerun.
9. Idempotency: read-only apart from the report.
10. Postconditions: `integrity-report@0.1.0`.
11. Verification: `chain_ok = true` and coverage equal for each triggered domain.
12. Failure routes: a broken link → `first_bad_id` reported, escalate; a coverage gap → escalate with the domain and counts.
13. Escalation triggers: any failure → owner and operator notice.
14. Evidence: report, last verified ID and hash (which the backup recipe may mirror off-site as an anchor).

**S9 — Apply retention (scheduled)**
1. Objective: keep the log within the approved retention period.
2. Capability: `nova.activity.retention.purge`.
3. Input mapping: `purge-request@0.1.0` with the approved `older_than_days`.
4. Dependencies: S8 passed for the range being removed.
5. Preconditions: retention policy approved; the first run was a reviewed dry run.
6. Action / side effects: as the retention role, delete entries older than the period in one transaction and append a tombstone recording the removed ID range and the last removed `entry_hash` (the chain continues from the tombstone).
7. Autonomy: A2 within the approved policy.
8. Timeout / retry: 5 minutes / 1 retry.
9. Idempotency: re-running removes nothing new.
10. Postconditions: `purge-result@0.1.0`; tombstone present.
11. Verification: no entry older than the period remains; chain verification from the tombstone passes.
12. Failure routes: S8 failed → skip the purge (never delete unverified history).
13. Escalation triggers: purge would remove more than 20 % of entries at once → ask the operator first.
14. Evidence: removed count, ID range, tombstone ID.

**S10 — Report**
1. Objective: an honest run record per path.
2. Capability: run record; jobs board (optional).
3. Input mapping: step results.
4. Dependencies: the last executed step.
5. Preconditions: none.
6. Action / side effects: emit the run envelope; update the jobs board row for verification and retention.
7. Autonomy: A2.
8. Timeout / retry: 5 s / 1 retry.
9. Idempotency: upsert.
10. Postconditions: run result emitted.
11. Verification: run status matches step outcomes.
12. Failure routes: board unavailable → run record only.
13. Escalation triggers: none.
14. Evidence: run ID.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Input validity | input matches `activity-input@0.1.0` | validator | valid | reject |
| Writer binding | entry `source` ∈ writer's sources | append log | 100 % | reject; notice on repeats |
| No secrets or payloads | post-redaction scan finds nothing; no JSON or base64 detail | S3 second scan | clean | reject |
| Append-only | UPDATE raises; DELETE only by the retention role | grant inventory, trigger tests | holds | stop, escalate as a security incident |
| Atomicity | a rolled-back domain change leaves no entry; a committed one has exactly one | rolled-back transaction tests, S8 coverage | holds | escalate |
| Chain | every `prev_hash` matches; tombstones bridge purges | S8 | `chain_ok` | escalate with `first_bad_id` |
| Coverage | triggered domain events = entries | S8 | equal | escalate |
| Feed scope | only the owner's session reads the feed; text escaped | access tests | holds | block the endpoint |
| Output schema | entries, pages and reports valid | validator | valid | fix; never fake |

A successful append proves that an entry exists, not that the logged action succeeded; the entry's `status` carries that.

## J. Self-correction and recovery

- **Failure classes:** validation, unauthenticated writer, unbound source, rate limit, redaction rejection, lock timeout, duplicate, chain break, coverage gap, retention conflict.
- **Permitted corrections:** truncate over-long text; drop unknown meta keys and non-relative links; return the existing ID for duplicates; retry gateway appends with the same key; append a correction entry.
- **Forbidden corrections:** edit, re-hash or delete entries to "repair" the chain; disable a trigger to unblock a domain write; let a writer append for an unbound source; store unredacted text.
- **Maximum correction cycles:** 1 per append; 1 rerun per verification.
- **Reverification required after correction:** yes — S3 second scan and S5 link check.
- **Stop conditions:** hard constraint fails; chain break detected (purges stop until resolved).
- **Recovery or compensation procedure:** correction entries; for a chain break, freeze purges, export the affected range for investigation, and restore from the latest verified backup if tampering is confirmed (owner decision).
- **What must be reported:** redactions by type, rejections by class, chain or coverage failures, purges.

## K. Independent challenge

- **Assumptions to challenge:** "no entry" means "no action" (it could mean a missing trigger or a writer that never calls the gateway); a `status: ok` from a writer reflects reality.
- **Alternative explanation:** a coverage gap may come from a domain table that changed shape (new status values the trigger map does not know).
- **Contradictory evidence:** the approval queue shows a consumed approval with no matching `email_sent` entry.
- **Could mislead:** a tidy feed that silently misses a whole source; redacted entries that hide what happened.
- **Separate verifier:** S8 recomputes the chain and counts domain events from the domain tables themselves, independently of the writers; the last verified hash can be mirrored off-site by the backup recipe so that even a database owner cannot rewrite history unnoticed.
- **Findings requiring disclosure:** coverage gaps, writers with no entries for 7 days, redaction trends.

## L. Human escalation

- **Triggers:** chain break; coverage gap; repeated redactions or auth failures from one writer; a purge larger than 20 %.
- **Why the agent cannot decide:** possible tampering, retention changes and writer permissions are owner or operator decisions.
- **Information presented:** affected IDs or domains, counts, first bad ID, the writer ID, the last good verification time.
- **Decision requested:** investigate and restore; fix the trigger map; revoke or fix a writer; confirm the purge.
- **Options and consequences:** revoking a writer stops its entries (its actions then show as coverage gaps if they have domain tables).
- **Safe behaviour while waiting:** keep appending; freeze purges; keep the feed readable.
- **If no response:** purges stay frozen; the jobs board shows the verification failure.
- **Resume:** a passing S8 run unfreezes purges.

## M. Completion conditions

Mandatory: entries valid and chained; append-only enforced; no secrets or payloads stored; writer binding enforced; atomicity with domain changes for triggers; verification reports truthful; evidence references present; limitations disclosed.

- `completed` — (append) entry stored or duplicate resolved; (feed) results returned; (verify) chain and coverage pass; (retention) purge done with tombstone.
- `partially_completed` — entry stored with redactions or truncation; feed returned with a narrowed window; verification passed the chain but found a coverage gap.
- `blocked` — writer unauthenticated or unbound, input invalid, or purge needs confirmation.
- `failed` — append transaction failed, chain verification failed, or retention could not complete.
- `cancelled` — the caller's transaction rolled back for its own reasons, or the operator stopped a purge.

A verification run with a coverage gap is never `completed`.

## N. Final output contract

Return per run: run ID; recipe ID and version; path (append, feed, verify, retention); status; one-line outcome; the path's output (`append-result`, `feed-page`, `integrity-report` or `purge-result`); step statuses; verification results and failed checks; evidence references (entry IDs, hashes, tombstone ID); approvals and scope (trigger installation, retention policy); side effects and reconciliation status; corrections; unresolved challenges; limitations; escalation state; learning candidate. Never include credentials, redacted text or payloads.

## O. Learning and revision

- **Outcome metrics:** entries per source per day, failed-status share, duplicate rate, redaction rate by writer, verification pass rate, coverage gaps, feed query latency.
- **Evidence retained:** integrity reports and tombstones, per retention policy.
- **Data minimisation:** metrics hold counts by source and writer, never titles or details.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}`.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review; changes to grants, the chain or retention need security review and cannot be applied by an automated repair.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (gateway `email_sent`) | entry stored, chained, visible in the feed | UNVERIFIED on Skill Port; gateway and feed observed in the reference implementation |
| Trigger path (booking inserted in a rolled-back test transaction) | entry visible inside the transaction, gone after rollback | UNVERIFIED on Skill Port; triggers observed in the reference implementation |
| Missing input (no `status`) | rejected | TBD |
| Invalid input (`ref_path` is an absolute URL) | link dropped; entry stored | TBD |
| Secret in detail (bearer token) | redacted, `redacted=true` | TBD |
| Payload in detail (JSON object) | rejected | TBD |
| Missing capability (logging function absent) | trigger installation refused; activation blocked | TBD |
| Permission denied (writer without credential) | 401, nothing stored | TBD |
| Unauthorized scope (mail writer logs a `voice` entry) | 403 | TBD |
| Tenant isolation | feed and verify never cross tenants | TBD |
| UPDATE or DELETE attempt by any writer role | refused | TBD |
| Timeout / lock contention | 503; retry with the same key stores once | TBD |
| Duplicate / replay (same `request_key`) | existing ID, `duplicate=true` | TBD |
| False success (writer logs `ok` for a failed send) | coverage or approval-outcome cross-check flags it | TBD |
| Failed verification (tampered row in a test copy) | `chain_ok=false`, `first_bad_id` reported | TBD |
| Coverage gap (trigger disabled in a test copy) | gap reported, run `partially_completed` | TBD |
| Correction budget exhausted | stops safely | TBD |
| Escalation (4 redactions from one writer) | operator notice | TBD |
| Cancellation (caller transaction rolls back) | no entry | TBD |
| Compensation (wrong entry) | correction entry appended; original unchanged | TBD |
| Retention purge | tombstone with range and last hash; chain verifies across it | TBD |
| Output schema and status | valid outputs; correct status | TBD |
| Compatibility / regression (new domain status value) | trigger map test fails before install, not in production | TBD |

All tests use rolled-back transactions or a test database copy; never persisted synthetic rows that a poller could act on. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, schema registry and whether the platform has its own audit log to reuse — TBD (owner: publisher, with Skill Port).
- Retention period — TBD (owner: operator, per local requirements).
- Same-transaction triggers mean a logging failure blocks the domain change; this is deliberate (no unlogged actions) but needs strong tests of the logging function.
- Writers that act outside the database (external sends) are logged only if they call the gateway; coverage for them relies on cross-checks (for example consumed approvals vs `email_sent` entries).
- Serialised appends per tenant cap throughput; adequate for a personal assistant, to be revisited for teams.
- Off-site anchoring of the last verified hash depends on the backup recipe (`nova.ops.backup-secret-scan-security-scan`).
