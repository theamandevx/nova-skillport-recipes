# Cost per job

## A. Identity and ownership

- **Recipe ID:** `nova.ops.cost-per-job` (proposed until assigned by Skill Port)
- **Name:** Cost per job
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** operations
- **Tags:** cost-tracking, ai-usage, tokens, attribution, high-water-mark, replay-guard, spend-alert
- **Summary:** Usage collectors read AI calls from the automation hub's execution records and the assistant's own session store, attribute tokens and spend to the job that caused them (`meta.job`), and append them to a usage ledger going forward only, with high-water marks, replay guards and honest cost bases. A daily check alerts the owner when spend exceeds about 2.5× the 7-day average.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** Read-only access to the automation hub's execution store (for example n8n) and a private copy of the assistant's session store; a usage ledger and watermark table in the same database; an operator-maintained price table with per-provider billing confirmation; a scheduler (every 15 minutes per collector, daily for the spike check); an owner notification channel. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — the core flow (going-forward collection with high-water marks, priced and attributed ledger rows, and the per-job view) runs in Nova's reference implementation; not every control described here has evidence yet (Section P lists the tests still needed). Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.usage.sources` | Capability | Read AI call records from each source, read-only | medium | UNVERIFIED |
| `nova.usage.sources.check_schema` | Operation (read) | Verify the source store still has the expected tables, primary key and fields | low | UNVERIFIED |
| `nova.usage.sources.read_new` | Operation (read) | Return call records after the watermark, bounded, with tokens, model, provider, time and job | medium | UNVERIFIED |
| `nova.usage.watermark` | Capability | Per-collector high-water marks | low | UNVERIFIED |
| `nova.usage.watermark.get` | Operation (read) | Current watermark, or none | low | UNVERIFIED |
| `nova.usage.watermark.rebaseline` | Operation (update, human-approved) | Move the watermark to the source's current maximum, skipping a gap (disclosed) | medium | UNVERIFIED |
| `nova.usage.pricing` | Capability | Price table with billing confirmation per provider tier | low | UNVERIFIED |
| `nova.usage.pricing.price` | Operation (read) | Price one record: billed cost, cost basis, list price | low | UNVERIFIED |
| `nova.usage.ledger` | Capability | Append-only usage ledger | medium | UNVERIFIED |
| `nova.usage.ledger.commit_batch` | Operation (create + update, one transaction) | Append rows (unique source reference) **and** advance the watermark together | medium | UNVERIFIED |
| `nova.usage.ledger.by_job` | Operation (read) | Cost and tokens per job over a window; unattributed rows per module | low | UNVERIFIED |
| `nova.usage.spike` | Capability | Daily spend-spike detection | low | UNVERIFIED |
| `nova.usage.spike.check` | Operation (read + alert) | Trailing 24 h spend vs the 7-day daily average, per job and in total | low | UNVERIFIED |
| `nova.notify.owner` | Capability | Deliver a message to the owner's channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Guard refusals, drift refusals and spike alerts | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`) of `check_schema` and `read_new`:**

- `.automation-hub` — reads execution records from the hub's store opened **read-only**, extracts token usage reported by AI-call steps in each execution, and attributes it as `meta.job = hub:<workflow name>`. Binding `binding://nova/usage/source-automation-hub`.
- `.assistant` — reads a **private copy** of the assistant's session store (never the live file) and attributes tokens as `meta.job = assistant:<scheduled job id>` or `assistant:chat`; cost is the assistant's own estimate with basis `assistant_estimate`. Binding `binding://nova/usage/source-assistant`.
- The two variants are complementary collectors, not fallbacks; each has its own watermark and runs on its own schedule.
- Every other operation supports only the default variant.

**Cost bases (field `cost_basis`):**

| Basis | `cost_usd` | When |
|---|---|---|
| `price_table` | tokens × confirmed price | provider tier billing confirmed by the operator |
| `assistant_estimate` | the assistant's own estimate | assistant source; labelled "estimate" in views |
| `unconfirmed_tier` | `0` | provider tier not confirmed as paid (for example responses indicate a free tier); `meta.list_price_usd` keeps the list price |
| `subscription_included` | `0` | calls covered by a flat subscription; `meta.list_price_usd` keeps the list price |

**Collector rules** (each exists because the naive alternative misstates spend):

- **C1 Going forward only:** the first run of a collector records a baseline watermark at the source's current maximum and emits nothing; no backfill.
- **C2 Idempotent high-water marks:** ledger rows and the watermark advance in **one transaction**; every row carries a unique (`source`, `source_ref`), so a crash between steps can never double-count.
- **C3 Replay guards:** if a run's batch exceeds `max_cost_usd_per_run` or `max_records_per_run`, the whole batch is refused: nothing written, watermark unchanged, owner alerted. This stops a lost watermark from replaying history into the ledger.
- **C4 Schema drift refuses:** if the source schema differs from the expected one (tables, primary key, fields), the run stops with `blocked` and emits nothing.
- **C5 Honest zero:** unconfirmed or subscription tiers are recorded at `0` with the list price kept separately; changing a tier to billed needs operator confirmation and applies going forward only.
- **C6 Single writer per call:** sources whose calls another writer already records are on an exclusion list; jobs whose scheduler saves no execution data are reported as `unmeasurable`, never as `0`.
- **C7 Attribution:** every row has `meta.job`; records that cannot be attributed get `<module>:unattributed`, so per-job totals reconcile with the overall total.
- **C8 Time:** each row's `ts` is the call time in UTC, clamped to now if it lies in the future.
- **C9 Metadata only:** the ledger stores tokens, model, provider, job and cost — never prompt or completion text.

**Bindings:** the two source bindings above plus `binding://nova/usage/watermark`, `binding://nova/usage/pricing`, `binding://nova/usage/ledger`, `binding://nova/usage/spike`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: two collector scripts on 15-minute system timers writing through a security-definer usage dispatcher (record and per-job read actions), with a per-job cost column on the jobs view.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Source store (hub executions, assistant session copy) | `schema://nova/cost-per-job/source-record@0.1.0` (normalised view) | the operator's estate | read only |
| Watermark | `schema://nova/cost-per-job/watermark@0.1.0` | one per collector | create (baseline), update (with commit or rebaseline) |
| Price table | `schema://nova/cost-per-job/price-table@0.1.0` | operator | read |
| Usage row | `schema://nova/cost-per-job/usage-record@0.1.0` | the operator's tenant ledger | create (append-only) |
| Spike alert | `schema://nova/cost-per-job/spike-check@0.1.0` | the owner | create |

**Authorization requirements (declared, never a grant):** `usage:sources:read` (`hub:executions:read`, `assistant:sessions:read_copy`), `usage:watermark:write` (collector identity), `usage:watermark:rebaseline` (human operator), `usage:pricing:read`, `usage:ledger:append`, `usage:ledger:read`, `notify:owner`. Source access through `secret_ref://usage-readonly`; the collector has no write access to any source and no update or delete on ledger rows.

**Policy / trust:** risk class `medium` (financial reporting; wrong numbers drive wrong decisions). Append-only ledger; replay guards; drift refusal; human approval for rebaseline and tier confirmation.

## C. Intent and applicability

- **User problem:** AI spend is spread over many automations and the assistant's own jobs, and the owner cannot see which job costs what or notice a runaway loop.
- **Desired outcome:** Each AI call becomes one attributed, honestly priced ledger row; per-job cost is visible; a spike triggers one clear alert.
- **Trigger or entry conditions:** Each collector's 15-minute schedule; the daily spike check; a manual run; a read of per-job cost.
- **Applicable situations:** LLM calls made by automation workflows and by the assistant (chat and scheduled jobs).
- **Out-of-scope situations:** Cloud infrastructure bills; media generation costs recorded by their own recipes at call time (for example `nova.content.post-image`); invoicing; changing models or budgets automatically.
- **Required inputs:** none for scheduled runs (Section D lists optional inputs).
- **Prerequisites:** Read-only source access; the ledger and watermark table; a price table with billing confirmation flags; guard values set or defaults accepted.
- **Required capabilities:** `nova.usage.sources`, `nova.usage.watermark`, `nova.usage.pricing`, `nova.usage.ledger`, `nova.usage.spike`, `nova.notify.owner` (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port. Provider-side billing exports for reconciliation are not assumed (`TBD`).
- **Assumptions:** The core flow runs in the reference implementation. Assumed: target sources record per-call token counts.
- **Ambiguity policy:** Use documented safe defaults: unknown tier → `unconfirmed_tier` at `0` with list price; unknown model → row recorded with `cost_basis = unconfirmed_tier` and flagged for pricing; unknown job → `<module>:unattributed`.
- **Vedaxi task type:** none — this recipe is not one of the seven task types Vedaxi supports today.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `collector` | enum `automation-hub`, `assistant` | yes | scheduler | — | Invalid → reject the run. |
| `trigger` | enum `schedule`, `manual` | yes | scheduler / operator | `schedule` | — |
| `mode` | enum `normal`, `rebaseline` | no | operator | `normal` | `rebaseline` without an approval ID → `blocked`. |
| `approval_id` | string | for `rebaseline` or a one-run guard raise | operator | — | Missing → `blocked`. |
| `guards` | object `{max_cost_usd_per_run, max_records_per_run}` | no | operator config | `TBD` (illustrative: 1.00 USD and 100 records per 15-minute run) | Missing → defaults; never unlimited. |
| `exclusions` | array of job names | no | operator config | empty | Unknown names ignored with a warning. |
| `now` | date-time | no | test harness only | system clock | Only honoured in test mode. |
| `dry_run` | boolean | no | operator / tests | `false` | `true` → compute, write nothing. |

```json
{"$id":"schema://nova/cost-per-job/run-request@0.1.0","type":"object","required":["collector","trigger"],"properties":{"collector":{"enum":["automation-hub","assistant"]},"trigger":{"enum":["schedule","manual"]},"mode":{"enum":["normal","rebaseline"]},"approval_id":{"type":"string"},"guards":{"type":"object","properties":{"max_cost_usd_per_run":{"type":"number","exclusiveMinimum":0},"max_records_per_run":{"type":"integer","minimum":1}}},"exclusions":{"type":"array","items":{"type":"string"}},"now":{"type":"string","format":"date-time"},"dry_run":{"type":"boolean"}}}
{"$id":"schema://nova/cost-per-job/schema-check@0.1.0","type":"object","required":["collector","expected","result"],"properties":{"collector":{"enum":["automation-hub","assistant"]},"expected":{"type":"object","required":["tables","primary_key","fields"],"properties":{"tables":{"type":"array","items":{"type":"string"}},"primary_key":{"type":"string"},"fields":{"type":"array","items":{"type":"string"}}}},"result":{"enum":["match","drift"]},"differences":{"type":"array","items":{"type":"string"}}}}
{"$id":"schema://nova/cost-per-job/source-record@0.1.0","type":"object","required":["source","source_ref","ts","provider","model","input_tokens","output_tokens","job"],"properties":{"source":{"enum":["automation-hub","assistant"]},"source_ref":{"type":"string"},"ts":{"type":"string","format":"date-time"},"provider":{"type":"string"},"model":{"type":"string"},"input_tokens":{"type":"integer","minimum":0},"output_tokens":{"type":"integer","minimum":0},"job":{"type":"string","pattern":"^(hub|assistant):.+$"},"assistant_estimate_usd":{"type":["number","null"],"minimum":0}}}
{"$id":"schema://nova/cost-per-job/watermark@0.1.0","type":"object","required":["collector","last_ref","updated_at","origin"],"properties":{"collector":{"enum":["automation-hub","assistant"]},"last_ref":{"type":"string"},"last_ts":{"type":["string","null"],"format":"date-time"},"updated_at":{"type":"string","format":"date-time"},"origin":{"enum":["baseline","commit","rebaseline"]},"approval_id":{"type":["string","null"]}}}
{"$id":"schema://nova/cost-per-job/price-table@0.1.0","type":"object","required":["as_of","entries"],"properties":{"as_of":{"type":"string","format":"date"},"entries":{"type":"array","items":{"type":"object","required":["provider","model","input_per_mtok_usd","output_per_mtok_usd","billing"],"properties":{"provider":{"type":"string"},"model":{"type":"string"},"input_per_mtok_usd":{"type":"number","minimum":0},"output_per_mtok_usd":{"type":"number","minimum":0},"billing":{"enum":["confirmed","unconfirmed","subscription_included"]}}}}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `run_result` | object `run-result@0.1.0` | `run_id`, `collector`, `status`, `rows_written`, `watermark_after` | run record | internal financial data | rows and sums re-read after commit (Section I) |
| `job_cost` | array of `job-cost@0.1.0` | per schema | jobs view, agent answers | internal | totals reconcile with the ledger total |
| `spike_check` | object `spike-check@0.1.0` | per schema | owner channel, run record | internal | recomputed from the ledger |

```json
{"$id":"schema://nova/cost-per-job/usage-record@0.1.0","type":"object","required":["usage_id","ts","source","source_ref","provider","model","units","cost_usd","cost_basis","meta"],"properties":{"usage_id":{"type":"string"},"ts":{"type":"string","format":"date-time"},"source":{"enum":["automation-hub","assistant"]},"source_ref":{"type":"string"},"provider":{"type":"string"},"model":{"type":"string"},"units":{"type":"object","required":["input_tokens","output_tokens"],"properties":{"input_tokens":{"type":"integer","minimum":0},"output_tokens":{"type":"integer","minimum":0}}},"cost_usd":{"type":"number","minimum":0},"cost_basis":{"enum":["price_table","assistant_estimate","unconfirmed_tier","subscription_included"]},"meta":{"type":"object","required":["job"],"properties":{"job":{"type":"string"},"list_price_usd":{"type":["number","null"],"minimum":0},"run_id":{"type":"string"}}}}}
{"$id":"schema://nova/cost-per-job/run-result@0.1.0","type":"object","required":["run_id","collector","status","rows_written","watermark_after"],"properties":{"run_id":{"type":"string"},"collector":{"enum":["automation-hub","assistant"]},"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},"outcome":{"enum":["baseline_set","rows_written","nothing_new","guard_refused","drift_refused","rebaselined","dry_run"]},"rows_read":{"type":"integer","minimum":0},"rows_written":{"type":"integer","minimum":0},"rows_excluded":{"type":"integer","minimum":0},"batch_cost_usd":{"type":"number","minimum":0},"batch_list_price_usd":{"type":"number","minimum":0},"watermark_before":{"type":["string","null"]},"watermark_after":{"type":["string","null"]},"unpriced_models":{"type":"array","items":{"type":"string"}}}}
{"$id":"schema://nova/cost-per-job/job-cost@0.1.0","type":"object","required":["job","window_days","cost_usd","input_tokens","output_tokens","measurable"],"properties":{"job":{"type":"string"},"window_days":{"type":"integer","minimum":1},"cost_usd":{"type":"number","minimum":0},"list_price_usd":{"type":"number","minimum":0},"input_tokens":{"type":"integer","minimum":0},"output_tokens":{"type":"integer","minimum":0},"bases":{"type":"array","items":{"type":"string"}},"measurable":{"type":"boolean"}}}
{"$id":"schema://nova/cost-per-job/spike-check@0.1.0","type":"object","required":["checked_at","ratio_threshold","results"],"properties":{"checked_at":{"type":"string","format":"date-time"},"ratio_threshold":{"type":"number","minimum":1},"floor_usd":{"type":"number","minimum":0},"results":{"type":"array","items":{"type":"object","required":["scope","last_24h_usd","avg_7d_usd","verdict"],"properties":{"scope":{"type":"string"},"last_24h_usd":{"type":"number","minimum":0},"avg_7d_usd":{"type":"number","minimum":0},"ratio":{"type":["number","null"]},"verdict":{"enum":["normal","spike","new_spend","insufficient_history"]},"alerted":{"type":"boolean"}}}}}}
{"$id":"schema://nova/cost-per-job/delivery@0.1.0","type":"object","required":["message_id","status"],"properties":{"message_id":{"type":"string"},"channel":{"type":"string"},"status":{"enum":["delivered","queued","failed"]},"delivered_at":{"type":["string","null"],"format":"date-time"}}}
```

**Example (fictional)**

Run output: `{"run_id":"usg_20260504_0915_hub","collector":"automation-hub","status":"completed","outcome":"rows_written","rows_read":14,"rows_written":12,"rows_excluded":2,"batch_cost_usd":0.041,"batch_list_price_usd":0.058,"watermark_before":"exec_88120","watermark_after":"exec_88134","unpriced_models":[]}`

Spike output: `{"checked_at":"2026-05-04T08:00:00Z","ratio_threshold":2.5,"floor_usd":0.50,"results":[{"scope":"hub:inbox-triage","last_24h_usd":1.92,"avg_7d_usd":0.31,"ratio":6.19,"verdict":"spike","alerted":true},{"scope":"total","last_24h_usd":2.40,"avg_7d_usd":1.05,"ratio":2.29,"verdict":"normal","alerted":false}]}` (amounts fictional).

## E. Constraints and budgets

- **Hard constraints:** read-only source access (read-only open or private copy); going forward only (C1); ledger rows and watermark in one transaction (C2); replay guards always set, never unlimited (C3); refuse on schema drift (C4); unconfirmed and subscription tiers at `0` with list price (C5); excluded sources never collected and unmeasurable jobs never shown as `0` (C6); every row attributed (C7); UTC timestamps clamped to now (C8); no prompt or completion text (C9); ledger append-only; rebaseline and tier changes only with human approval.
- **Soft constraints:** finish each run in under 10 seconds; label estimates as estimates in every view.
- **Resource and tenant scope:** the operator's estate and ledger.
- **Data sources allowed:** the configured source stores, the price table, the ledger.
- **External transfers permitted:** none beyond owner alerts (job names and amounts only).
- **Time limit:** collector run ≤ 60 s; spike check ≤ 30 s.
- **Cost or token budget:** no LLM calls (deterministic extraction and pricing).
- **Maximum steps:** 11 (S1–S11).
- **Maximum retries and correction cycles:** source reads: 1 retry; commit: 1 retry (idempotent); the next scheduled run is the natural retry.
- **Rate limits and concurrency:** one run per collector at a time; batch ≤ `max_records_per_run`.
- **Retention and logging constraints:** ledger retention `TBD` (operator); logs carry counts and sums only.
- **Prohibited actions:** editing or deleting ledger rows; backfilling history; writing to sources; raising a guard without approval; flipping a tier to billed without confirmation.
- **Stop conditions:** schema drift; guard exceeded; watermark table unavailable; another run holds the lock.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Usage rows | guards pass; schema matches | Allowed | rows re-read by `source_ref` |
| Create | Baseline watermark | first run for a collector | Allowed | watermark readable, origin `baseline` |
| Read | Source stores, price table, ledger | read-only handles | Allowed | handle mode recorded |
| Update | Watermark (`last_ref`, `last_ts`, `updated_at`, `origin`) | in the same transaction as the rows | Allowed | equals the batch's last reference |
| Update | Watermark by rebaseline | operator approval ID | Approval Required | origin `rebaseline` with approval ID |
| Delete | any ledger row or watermark | — | Not Allowed | — |
| External side effect | owner alerts (guard, drift, spike) | condition met; deduplicated | Allowed | delivery receipt |

- **Idempotency:** unique (`source`, `source_ref`) on usage rows; watermark advanced in the same transaction; spike alerts keyed by (scope, date).
- **Expected side effects:** ledger rows appended; watermark moves forward; occasional alerts.
- **Compensation:** a wrong price is corrected by appending adjustment rows (`meta.adjusts_usage_id`) after operator approval — never by editing.
- **Irreversible:** alerts sent; a rebaseline skips a window permanently (disclosed).
- **Ambiguous outcomes:** if the commit times out, re-read the watermark: advanced → done; unchanged → re-run the batch (unique keys prevent duplicates).

## G. Autonomy and approval policy

- **Default autonomy level:** A3 — conditional autonomy for collection and reporting.
- **Planning level:** A3.
- **Execution level:** A3 within guards; refusal and escalation outside them.
- **Verification level:** A3.
- **Recovery level:** A1 for rebaseline, guard raises and tier changes; A3 for retries.
- **Approval required for:** rebaseline; a one-run guard raise to ingest a genuine burst; confirming a tier as billed; price corrections via adjustment rows.
- **Actions never permitted:** backfill; editing or deleting rows; collecting excluded sources; recording prompt text; changing models, budgets or routing.
- **Conditions requiring clarification:** an unpriced model appears (operator adds a price).
- **Conditions requiring escalation:** guard refusal; schema drift; spike alert; watermark unreadable.
- **Safe state after stop, cancellation or escalation:** nothing written for the refused batch; watermark unchanged; sources untouched.

## H. Execution plan

**S1 — Start the run**
1. Objective: one run per collector at a time.
2. Capability: run lock in `nova.usage.watermark`.
3. Input mapping: `run-request@0.1.0`.
4. Dependencies: none.
5. Preconditions: schedule fired or manual request.
6. Action / side effects: take the collector lock.
7. Autonomy: A3.
8. Timeout / retry: 5 s / none (skip if locked).
9. Idempotency: lock per collector.
10. Postconditions: lock held.
11. Verification: lock owner = this run.
12. Failure routes: locked → `cancelled` (skipped); stale lock > 15 minutes → take over and record.
13. Escalation triggers: none.
14. Evidence: `run_id`, lock record.

**S2 — Schema-drift check**
1. Objective: refuse to read a store whose shape changed.
2. Capability: `nova.usage.sources.check_schema` (`.automation-hub` or `.assistant`).
3. Input mapping: expected schema for the collector.
4. Dependencies: S1.
5. Preconditions: read-only handle (or fresh private copy for the assistant).
6. Action: compare tables, primary key and fields.
7. Autonomy: A3.
8. Timeout / retry: 10 s / 1 retry.
9. Idempotency: read-only.
10. Postconditions: `schema-check@0.1.0` = `match`.
11. Verification: every expected field present; primary key unchanged.
12. Failure routes: `drift` → `blocked` (`drift_refused`), nothing emitted.
13. Escalation: alert the operator with the differences.
14. Evidence: schema-check result.

**S3 — Load the watermark**
1. Objective: know where the last run stopped.
2. Capability: `nova.usage.watermark.get`.
3. Input mapping: `collector`.
4. Dependencies: S2.
5. Preconditions: schema matches.
6. Action: read; if none, set a baseline at the source's current maximum and end with `baseline_set` (C1).
7. Autonomy: A3.
8. Timeout / retry: 5 s / 1 retry.
9. Idempotency: baseline insert is conditional on no existing watermark.
10. Postconditions: watermark known, or baseline set.
11. Verification: watermark ≤ source maximum.
12. Failure routes: watermark ahead of the source maximum (source reset) → `blocked`, escalate (needs rebaseline).
13. Escalation: as above.
14. Evidence: watermark before.

**S4 — Read new records**
1. Objective: the calls made since the watermark.
2. Capability: `nova.usage.sources.read_new`.
3. Input mapping: watermark, `max_records_per_run` + 1 (to detect overflow).
4. Dependencies: S3.
5. Preconditions: watermark known.
6. Action: read ordered by source reference; normalise to `source-record@0.1.0`; drop records from excluded jobs (count them).
7. Autonomy: A3.
8. Timeout / retry: 20 s / 1 retry.
9. Idempotency: read-only.
10. Postconditions: batch of records.
11. Verification: references strictly greater than the watermark; no prompt text kept.
12. Failure routes: read error → `failed`; next run retries.
13. Escalation: none.
14. Evidence: rows read, rows excluded.

**S5 — Extract and attribute**
1. Objective: tokens and job per record.
2. Capability: extraction within `read_new`.
3. Input mapping: raw call data.
4. Dependencies: S4.
5. Preconditions: records present (else `nothing_new`).
6. Action: take input and output tokens, provider and model from the call's reported usage; set `job` from the workflow name or scheduled job ID; else `<module>:unattributed`; normalise `ts` to UTC and clamp to now.
7. Autonomy: A3.
8. Timeout / retry: 5 s / none.
9. Idempotency: pure transformation.
10. Postconditions: attributed records.
11. Verification: every record has a job; token counts ≥ 0.
12. Failure routes: unreadable usage in one record → keep it with zero tokens and flag (`unpriced`), never drop silently.
13. Escalation: none.
14. Evidence: attribution counts.

**S6 — Price each record**
1. Objective: honest cost and basis.
2. Capability: `nova.usage.pricing.price`.
3. Input mapping: provider, model, tokens, source.
4. Dependencies: S5.
5. Preconditions: price table loaded.
6. Action: `confirmed` → `price_table`; `unconfirmed` → `0` with `list_price_usd`; `subscription_included` → `0` with list price; assistant source → `assistant_estimate`; unknown model → `unconfirmed_tier` with list price null, reported in `unpriced_models`.
7. Autonomy: A3.
8. Timeout / retry: 5 s / none.
9. Idempotency: pure.
10. Postconditions: priced rows.
11. Verification: `cost_usd = 0` whenever basis is `unconfirmed_tier` or `subscription_included`.
12. Failure routes: price table missing → `blocked`.
13. Escalation: unpriced models → ask the operator to add prices.
14. Evidence: batch cost and list price.

**S7 — Replay guard**
1. Objective: never ingest an implausible batch.
2. Capability: guard check (local).
3. Input mapping: batch size, batch cost (billed plus list price for zero-cost bases), guards.
4. Dependencies: S6.
5. Preconditions: priced batch.
6. Action: refuse if records > `max_records_per_run` or cost > `max_cost_usd_per_run`.
7. Autonomy: A3.
8. Timeout / retry: 1 s / none.
9. Idempotency: pure.
10. Postconditions: pass, or refused.
11. Verification: comparison recorded with the numbers.
12. Failure routes: refused → `blocked` (`guard_refused`), nothing written, watermark unchanged.
13. Escalation: alert the owner with counts, cost and the source reference range; options: rebaseline (skip) or a one-run guard raise (approval).
14. Evidence: guard decision.

**S8 — Commit rows and advance the watermark**
1. Objective: append the batch and move the watermark atomically.
2. Capability: `nova.usage.ledger.commit_batch`.
3. Input mapping: priced rows, last source reference.
4. Dependencies: S7 passed.
5. Preconditions: not `dry_run`.
6. Action / side effects: one transaction: insert rows (unique `source`, `source_ref`), update watermark (origin `commit`).
7. Autonomy: A3.
8. Timeout / retry: 10 s / 1 retry.
9. Idempotency: unique keys plus same-transaction watermark.
10. Postconditions: rows and watermark committed together.
11. Verification: see S9.
12. Failure routes: transaction fails → rolled back; `failed`; next run retries.
13. Escalation: 3 consecutive failures → operator alert.
14. Evidence: transaction ID, rows inserted.

**S9 — Verify the commit**
1. Objective: prove the ledger holds exactly this batch.
2. Capability: `nova.usage.ledger.by_job` (read by `run_id`).
3. Input mapping: `run_id`.
4. Dependencies: S8.
5. Preconditions: committed.
6. Action: re-read rows for the run; compare count and cost sum; confirm the watermark equals the batch's last reference.
7. Autonomy: A3.
8. Timeout / retry: 5 s / 1 retry.
9. Idempotency: read-only.
10. Postconditions: verified.
11. Verification: counts and sums equal; no duplicate `source_ref`.
12. Failure routes: mismatch → `partially_completed`, flag for operator review.
13. Escalation: mismatch → alert.
14. Evidence: verification result.

**S10 — Daily spike check (scheduled once a day)**
1. Objective: catch runaway spend early.
2. Capability: `nova.usage.spike.check`.
3. Input mapping: ledger daily totals per job and overall; ratio 2.5; floor `TBD` (illustrative 0.50 USD); minimum history 3 days.
4. Dependencies: ledger rows (any runs).
5. Preconditions: not run yet today.
6. Action: for each job and the total, compare trailing-24 h billed spend with the mean daily spend of the previous 7 full days; `spike` when ratio ≥ 2.5 and the increase ≥ floor; `new_spend` when the average is 0 and spend ≥ floor; `insufficient_history` under 3 days. Zero-cost bases are checked on list price as an informational "list-price spike".
7. Autonomy: A3.
8. Timeout / retry: 30 s / 1 retry.
9. Idempotency: alert key (scope, date).
10. Postconditions: `spike-check@0.1.0`.
11. Verification: numbers recomputed directly from ledger rows.
12. Failure routes: ledger unreadable → `failed`; retry next day, operator told.
13. Escalation: spikes → owner alert naming the job, both amounts and the ratio.
14. Evidence: spike-check result.

**S11 — Report and release**
1. Objective: an auditable run record.
2. Capability: run record; `nova.notify.owner.send` for pending alerts.
3. Input mapping: `run_result`.
4. Dependencies: S9 or an earlier terminal step.
5. Preconditions: run finished.
6. Action / side effects: write the run record; send queued guard or drift alerts; release the lock.
7. Autonomy: A3.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: by `run_id`; alerts send-then-mark.
10. Postconditions: run recorded; lock released.
11. Verification: run record readable.
12. Failure routes: alert channel down → keep alerts pending for the next run.
13. Escalation: none.
14. Evidence: `run_result`, message IDs.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Read-only sources | handles read-only or private copy | S2 | always | abort; escalate |
| Schema match | source shape = expected | S2 | match | `blocked` |
| Going forward | first run writes no rows | S3 | zero rows | treat as defect |
| No duplicates | unique `source_ref` per source | ledger constraint, S9 | none | investigate |
| Atomic watermark | watermark = last committed reference | S9 | equal | `partially_completed` |
| Guard | batch within guards | S7 | pass | refuse |
| Honest zero | zero-cost rows carry a basis and list price | S6 | all | fix before commit |
| Attribution | every row has `meta.job` | S5 | 100 % | `unattributed` |
| Reconciliation | sum of per-job cost = ledger total for the window | `by_job` | equal | flag |
| No content | no prompt or completion text in rows or logs | S4, S8 | none | stop; purge the run's rows with approval |
| Output schema | outputs valid | validator | valid | fix fields; never fake values |

A successful insert proves only that rows exist; S9's re-read and the reconciliation prove the numbers are right.

## J. Self-correction and recovery

- **Failure classes:** lock contention, schema drift, source unavailable, watermark ahead of source, unpriced model, guard exceeded, commit failure, verification mismatch, alert delivery failure.
- **Permitted corrections:** one retry of reads and commits; re-run after an ambiguous commit (idempotent); take over a stale lock; queue alerts.
- **Forbidden corrections:** raising guards or rebaselining without approval; editing rows; backfilling; pricing an unconfirmed tier as billed; dropping records to fit a guard.
- **Maximum correction cycles:** 1 per step per run.
- **Reverification required after correction:** yes — S9 after any re-run.
- **Stop conditions:** drift, guard refusal, watermark ahead of source.
- **Recovery or compensation:** approved rebaseline (gap disclosed) or one-run guard raise; approved adjustment rows for price fixes.
- **What must be reported:** rows written, refusals with reasons, unpriced models, gaps created by rebaselines.

## K. Independent challenge

- **Assumptions to challenge:** the source reports every AI call; the price table is current; "free tier" really costs nothing.
- **Alternative approach:** reconcile monthly against provider invoices or billing exports where available (`TBD`).
- **Contradictory evidence:** provider dashboards showing spend for a tier recorded as `unconfirmed_tier`; tokens without a matching job.
- **Could mislead:** a per-job view omitting unmeasurable jobs; an estimate shown as billed; a total that hides unattributed spend.
- **Separate verifier:** S9 re-read and the reconciliation of per-job totals against the ledger total; spike numbers recomputed from raw rows.
- **Findings requiring disclosure:** unmeasurable jobs, excluded sources, estimates, unconfirmed tiers with list price, rebaseline gaps.

## L. Human escalation

- **Triggers:** guard refusal; schema drift; watermark ahead of source; unpriced models; spike or new-spend alert; repeated commit failures.
- **Why the agent cannot decide:** skipping data, raising limits and confirming billing are financial decisions.
- **Information presented:** collector, counts, batch cost and list price, reference range, schema differences, spike numbers per job.
- **Decision requested:** rebaseline, one-run guard raise, add prices, confirm a tier as billed, or investigate the spiking job.
- **Options and consequences:** rebaseline loses the window's attribution; a guard raise ingests it as-is.
- **Safe behaviour while waiting:** later runs keep refusing the same batch; nothing written; watermark unchanged.
- **If no response:** the refusal alert repeats once a day, not every run.
- **Resume:** with an approval ID on the next run; S2 re-checks the schema first.

## M. Completion conditions

Mandatory: outputs valid; rows and watermark committed atomically or nothing written; guards and drift checks applied; every row attributed and honestly priced; verification passed; limitations disclosed; evidence references present.

- `completed` — baseline set, nothing new, or rows written and verified; spike check run when due.
- `partially_completed` — rows written but verification found a mismatch, or alerts are pending delivery.
- `blocked` — schema drift, guard refusal, watermark ahead of source, missing price table or missing approval.
- `failed` — source read or commit failed and was rolled back.
- `cancelled` — skipped because another run held the lock, or stopped by the operator.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome (for example "12 rows, 0.041 USD billed, 0.058 USD list"); `run_result` (and `spike_check` when run); step statuses S1–S11; verification results and failed checks; evidence references (schema check, watermark before and after, transaction ID); approvals and scope; side effects (rows, watermark, alerts) and reconciliation status; corrections; unresolved challenges; limitations (estimates, unmeasurable jobs); escalation state; learning candidate. Never include prompts, completions or credentials.

## O. Learning and revision

- **Outcome metrics:** guard refusal rate, drift refusals, unattributed share, unmeasurable job count, estimate-vs-invoice delta (where available), spike precision (alerts that led to action).
- **Evidence retained:** run records, ledger rows, spike results.
- **Data minimisation:** token counts and costs only; no content.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}`.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review; guards can never be removed and the ledger can never become editable through an automated revision.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (fixture source with 12 new calls) | 12 attributed, priced rows; watermark advanced; verified | UNVERIFIED on Skill Port; observed in the reference implementation |
| First run baseline | watermark set at source maximum; zero rows | UNVERIFIED on Skill Port; observed in the reference implementation |
| Invalid input (`rebaseline` without approval) | `blocked` | TBD |
| Missing input (no guards configured) | defaults applied; never unlimited | TBD |
| Missing capability (assistant source binding absent) | `blocked` with a dependency message | TBD |
| Permission denied (source unreadable) | `failed`; nothing written | TBD |
| Unauthorized scope (collector attempts a row update) | refused by the ledger | TBD |
| Tenant isolation | tenant A's ledger invisible to tenant B | TBD |
| Schema drift (primary key renamed) | `blocked`, `drift_refused`, zero rows | TBD |
| Timeout during commit | re-read watermark; re-run writes no duplicates | TBD |
| Duplicate / replay (watermark lost) | guard refuses the replayed history | TBD |
| Unconfirmed tier | `cost_usd = 0`, list price kept | TBD |
| Excluded source | not collected; counted as excluded | TBD |
| False success (insert reports success, rows missing) | S9 mismatch → `partially_completed` | TBD |
| Failed verification (per-job sum ≠ total) | flagged | TBD |
| Spike check (6× average above floor; 2× average; 2 days history) | `spike` alert; `normal`; `insufficient_history` | TBD |
| Correction budget exhausted (commit fails 3 runs) | escalation | TBD |
| Escalation (guard refusal with approval to raise once) | next run ingests the batch, guard restored | TBD |
| Cancellation (lock held) | `cancelled`, nothing written | TBD |
| Rollback (failure mid-transaction) | no rows, watermark unchanged | TBD |
| Output schema and status | valid outputs, correct status | TBD |
| Compatibility / regression (source version adds fields) | still `match` on required fields; extraction unchanged | TBD |

All tests use fixture stores and a sandbox ledger; never production financial data. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, schema registry and host-level read-only access — TBD (owner: publisher, with Skill Port).
- Guard values, spike floor, ledger retention and price-table maintenance — TBD (owner: operator).
- Jobs whose scheduler saves no execution data cannot be measured by this recipe.
- Assistant costs are the assistant's own estimates until reconciled against invoices.
- Provider invoice reconciliation — TBD (owner: operator).
