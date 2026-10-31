# Job health monitor

## A. Identity and ownership

- **Recipe ID:** `nova.ops.job-health-monitor` (proposed until assigned by Skill Port)
- **Name:** Job health monitor
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** operations
- **Tags:** monitoring, scheduled-jobs, job-registry, honest-status, stale-detection, alerting
- **Summary:** Every 5 minutes, the agent discovers every recurring job across the automation hub, the system timers and the assistant's own scheduler, derives one honest state per job from read-only run evidence, writes it to a single registry and alerts the owner only when a state changes. A registry row older than 20 minutes is shown as `unknown`, never as healthy.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** Read-only access to the automation hub's workflow and execution store (for example n8n), to system timer and service properties and journal (for example systemd), and to the assistant's job definitions and execution history; optionally a database cron history; a registry table with a dispatcher function; a scheduler that runs the collector every 5 minutes; an owner notification channel. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — the core flow (discovery across the schedulers, state derivation, and the registry with its view) runs in Nova's reference implementation; not every control described here has evidence yet (Section P lists the tests still needed). Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.jobs.sources` | Capability | Read job definitions and run evidence from each scheduler, read-only | medium | UNVERIFIED |
| `nova.jobs.sources.discover` | Operation (read) | List every recurring job a scheduler knows, with trigger type and expected cadence | low | UNVERIFIED |
| `nova.jobs.sources.collect_runs` | Operation (read) | Run outcomes in the window, last run, last success, next run, item counts, last error | medium | UNVERIFIED |
| `nova.jobs.evaluate` | Capability | Turn evidence into one state per job | low | UNVERIFIED |
| `nova.jobs.evaluate.derive_state` | Operation (pure function) | Apply the state rules below; return state and reason | low | UNVERIFIED |
| `nova.jobs.registry` | Capability | One row per job: definition plus live state | low | UNVERIFIED |
| `nova.jobs.registry.report` | Operation (create/update, collector only) | Upsert live columns for all jobs in one transaction | low | UNVERIFIED |
| `nova.jobs.registry.record_cycle` | Operation (create/update) | Open and close a cycle record with a single-flight lock | low | UNVERIFIED |
| `nova.jobs.registry.list` | Operation (read) | Rows with `display_state` (staleness applied at read time) | low | UNVERIFIED |
| `nova.jobs.registry.summary` | Operation (read) | Counts per state, stale banner flag, badge colour | low | UNVERIFIED |
| `nova.jobs.registry.set` | Operation (update, human only) | Note, lane, `off_on_purpose` | low | UNVERIFIED |
| `nova.notify.owner` | Capability | Deliver a message to the owner's channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Send state-change alerts | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`) of `discover` and `collect_runs`:**

- `.automation-hub` — reads workflows (active flag, trigger type, interval) and executions from the hub's store opened **read-only** (for example a read-only database URI), with bounded queries over the window; decodes the store's serialisation format to extract error messages instead of pattern-matching serialised blobs. Binding `binding://nova/jobs/source-automation-hub`.
- `.system-timers` — reads timer and service properties (enabled, active, sub-state, next elapse time, last trigger) and journal entries for each unit. Binding `binding://nova/jobs/source-system-timers`.
- `.assistant-scheduler` — reads the assistant's job definitions (schedule, last status, last error, failure streak, next run) and a **private copy** of its execution history, never the live file. Binding `binding://nova/jobs/source-assistant-scheduler`.
- `.database-cron` — optional; reads a database cron job list and run details. Binding `binding://nova/jobs/source-database-cron`.
- These variants are **not fallbacks** for each other: each covers different jobs, all run every cycle, and a failing variant leaves its jobs' rows un-refreshed (so they turn `unknown` after 20 minutes) rather than dropping or greening them.
- `derive_state`, `report`, `record_cycle`, `list`, `summary`, `set`, `send` — only the default variant is supported.

**States** (stored `state`; plus a read-time `display_state`):

| State | Meaning |
|---|---|
| `running` | Running as expected: on schedule and succeeding (includes a timer whose service is executing right now) |
| `late` | An expected run did not happen on time |
| `failing` | A real outage: in the failure window, runs were attempted and none succeeded, or more than 25 % errored with more than 3 errors |
| `no_work` | Succeeding on schedule, but every run in the window reported zero work items (only where the source exposes item counts) |
| `on_demand` | Webhook- or event-triggered, with no expected cadence; can never be `late` |
| `not_measured` | The scheduler keeps no run records for this job; shown deliberately so its absence from the counts is visible |
| `off_on_purpose` | Disabled intentionally by a human, with a note |
| `unknown` (display only) | The row is older than 20 minutes or its source failed; never shown as healthy |

**State rules** (each exists because the naive alternative produces a false state):

- **R1 Stale ≠ healthy:** at read time, a row whose `collected_at` is older than 20 minutes displays `unknown`, and the view shows a stale banner.
- **R2 Two timestamps:** `last_run_at` and `last_success_at` are separate fields; a job that runs every 15 minutes but last succeeded days ago is never green.
- **R3 Failing means an outage:** `failing` only when, in the window (24 h, or one expected interval plus grace for jobs whose cadence is longer than 24 h), attempts ≥ 1 and successes = 0, **or** errors ÷ attempts > 0.25 **and** errors > 3. One error in many hundreds of runs stays `running`.
- **R4 Timers by the scheduler's own clock:** enabled and timer inactive → `late`; `failing` stays `failing`; service currently executing (sub-state running, next-run blank) → `running`; otherwise `running` if and only if the scheduler's next-run time is in the future, else `late`. This handles windows, weekdays and months without parsing calendar expressions. Sources that expose no next-run time fall back to elapsed time: `late` when now − `last_run_at` > 2 × expected interval + 5 minutes (`TBD` per operator).
- **R5 On-demand never late:** jobs with no expected interval cannot be `late`; they can be `failing`.
- **R6 Not measured is explicit:** jobs whose scheduler persists no run records are `not_measured`; jobs whose scheduler discards successful runs are judged on errors only and are never `running` by inference (`not_measured` unless failing).
- **R7 One marker per outcome:** count exactly one canonical marker per run (for example a oneshot service logs two lines on success and two on failure; count only "finished" and "failed with result").
- **R8 Normalised time:** every timestamp is converted to UTC with an explicit offset before comparison (stores mix naive UTC and offset-aware values); the scheduler's formatted timestamps are parsed as well as epoch values; an unparseable timestamp yields `unknown` with a reason, never `running`.
- **R9 Pollers held out:** every-minute pollers carry lane `poller` and are excluded from headline run volume.
- **R10 Human-only off:** only a human sets `off_on_purpose`; if such a job runs again it is flagged for review.

**Bindings:** the four source bindings above plus `binding://nova/jobs/evaluate`, `binding://nova/jobs/registry`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: a collector script on a 5-minute system timer writing through a security-definer registry dispatcher (`list`, `summary`, `report`, `set`), with a dashboard page and a navigation badge.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Scheduler stores (hub, timers, assistant, database cron) | source-native | the operator's estate | read only |
| Job state row | `schema://nova/job-health-monitor/job-state@0.1.0` | the operator's tenant | create, read, update (live columns by collector; human columns by `set`) |
| Cycle record | `schema://nova/job-health-monitor/cycle-result@0.1.0` | the operator's tenant | create, update |
| Alert | `schema://nova/job-health-monitor/alert@0.1.0` | the owner | create |

**Authorization requirements (declared, never a grant):** `jobs:sources:read` (per source: `hub:executions:read`, `timers:read`, `journal:read`, `assistant:jobs:read`, `dbcron:read`), `jobs:registry:report` (collector identity), `jobs:registry:read`, `jobs:registry:set` (human operators only), `notify:owner`. Source access uses `secret_ref://jobwatch-readonly` style references; no write credential to any scheduler store is ever issued to the collector.

**Policy / trust:** risk class `medium` (broad read access to operational data; misleading status could hide outages). Read-only on every scheduler; never restarts, re-runs, enables or disables a job.

## C. Intent and applicability

- **User problem:** Dozens of recurring jobs run across several schedulers, and nobody can answer "is everything running?" — a silent failure can go unnoticed.
- **Desired outcome:** One registry and view where every job appears automatically with an honest state, and the owner hears only about real changes.
- **Trigger or entry conditions:** The 5-minute schedule; a manual "refresh now"; a read of the registry (agent question or dashboard).
- **Applicable situations:** Workflow automations, system timers, the assistant's scheduled tasks, database cron jobs.
- **Out-of-scope situations:** Restarting or fixing jobs (a human or a separate remediation recipe does that); application-level monitoring (latency, uptime of web endpoints); cost per job (see `nova.ops.cost-per-job`, which can add a cost column).
- **Required inputs:** none for scheduled cycles; optional filters for reads (Section D).
- **Prerequisites:** Read-only access to each scheduler source; the registry and its dispatcher; a scheduler for the collector; operator thresholds set or defaults accepted.
- **Required capabilities:** `nova.jobs.sources`, `nova.jobs.evaluate`, `nova.jobs.registry`, `nova.notify.owner` (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port. An external heartbeat that alerts when the collector itself stops needs a scheduler independent of the collector.
- **Assumptions:** The core flow runs in the reference implementation. Assumed: target schedulers expose equivalent read-only evidence.
- **Ambiguity policy:** Use documented safe defaults: when evidence is missing or contradictory, the state is `unknown` or `not_measured` — never `running`.
- **Vedaxi task type:** none — this recipe is not one of the seven task types Vedaxi supports today.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `trigger` | enum `schedule`, `manual` | yes | scheduler / operator | `schedule` | Invalid → reject the cycle. |
| `sources` | array of enum `automation-hub`, `system-timers`, `assistant-scheduler`, `database-cron` | no | operator | all configured | Unknown source → ignored with a warning. |
| `thresholds` | object (below) | no | operator config | defaults below | Out of range → defaults with a warning. |
| `now` | date-time | no | test harness only | system clock | Only honoured in test mode. |
| `dry_run` | boolean | no | operator / tests | `false` | `true` → compute and return, write nothing, send nothing. |
| `read_filter` | object `{state?, source?, lane?}` | no | reader | none | Unknown values → empty result. |

```json
{"$id":"schema://nova/job-health-monitor/cycle-request@0.1.0","type":"object","required":["trigger"],
 "properties":{"trigger":{"enum":["schedule","manual"]},
 "sources":{"type":"array","items":{"enum":["automation-hub","system-timers","assistant-scheduler","database-cron"]},"uniqueItems":true},
 "thresholds":{"type":"object","properties":{"stale_after_min":{"type":"integer","minimum":10,"maximum":60,"default":20},
  "failure_window_h":{"type":"integer","minimum":1,"maximum":168,"default":24},"error_ratio":{"type":"number","minimum":0.05,"maximum":0.9,"default":0.25},
  "min_errors_exclusive":{"type":"integer","minimum":0,"default":3},"elapsed_tolerance_factor":{"type":"number","minimum":1,"default":2}}},
 "now":{"type":"string","format":"date-time"},"dry_run":{"type":"boolean"}}}
```

```json
{"$id":"schema://nova/job-health-monitor/job-state@0.1.0","type":"object",
 "required":["job_id","source","name","lane","state","state_reason","collected_at"],
 "properties":{"job_id":{"type":"string","pattern":"^(hub|timer|assistant|dbcron):.+$"},
 "source":{"enum":["automation-hub","system-timers","assistant-scheduler","database-cron"]},"name":{"type":"string"},
 "lane":{"enum":["scheduled","poller","on_demand"]},"enabled":{"type":"boolean"},"expected_interval_s":{"type":["integer","null"]},
 "next_run_at":{"type":["string","null"],"format":"date-time"},"last_run_at":{"type":["string","null"],"format":"date-time"},
 "last_success_at":{"type":["string","null"],"format":"date-time"},"runs_window":{"type":["integer","null"]},
 "errors_window":{"type":["integer","null"]},"items_window":{"type":["integer","null"]},
 "state":{"enum":["running","late","failing","no_work","on_demand","not_measured","off_on_purpose"]},
 "display_state":{"enum":["running","late","failing","no_work","on_demand","not_measured","off_on_purpose","unknown"]},
 "state_reason":{"type":"string","maxLength":200},"error_excerpt":{"type":["string","null"],"maxLength":300},
 "collected_at":{"type":"string","format":"date-time"},"present":{"type":"boolean"},"note":{"type":["string","null"]}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `cycle_result` | object `schema://nova/job-health-monitor/cycle-result@0.1.0` | `cycle_id`, `status`, `sources`, `counts`, `transitions` | registry cycle record; run record | internal operational data | counts reconcile with written rows (Section I) |
| `job_list` | array of `job-state@0.1.0` with `display_state` | per schema | dashboard, agent answers | internal | staleness rule applied at read time |

```json
{"$id":"schema://nova/job-health-monitor/cycle-result@0.1.0","type":"object","required":["cycle_id","status","sources","counts","transitions"],
 "properties":{"cycle_id":{"type":"string"},"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},
 "started_at":{"type":"string","format":"date-time"},"finished_at":{"type":"string","format":"date-time"},
 "sources":{"type":"array","items":{"type":"object","required":["source","result"],"properties":{"source":{"type":"string"},
  "result":{"enum":["ok","error","suspect","skipped"]},"jobs_discovered":{"type":"integer"},"reason":{"type":"string"}}}},
 "counts":{"type":"object","additionalProperties":{"type":"integer","minimum":0}},
 "transitions":{"type":"array","items":{"type":"object","required":["job_id","from","to"],"properties":{"job_id":{"type":"string"},"from":{"type":["string","null"]},"to":{"type":"string"}}}},
 "alerts_sent":{"type":"integer","minimum":0}}}
```

Supporting schemas:

```json
{"$id":"schema://nova/job-health-monitor/source-evidence@0.1.0","type":"object","required":["source","result","jobs"],"properties":{"source":{"enum":["automation-hub","system-timers","assistant-scheduler","database-cron"]},"result":{"enum":["ok","error"]},"handle_mode":{"enum":["read_only","private_copy"]},"jobs":{"type":"array","items":{"type":"object","required":["native_id","name","trigger"],"properties":{"native_id":{"type":"string"},"name":{"type":"string"},"trigger":{"enum":["schedule","poller","webhook","event"]},"enabled":{"type":"boolean"},"expected_interval_s":{"type":["integer","null"]},"next_run_at":{"type":["string","null"]},"sub_state":{"type":["string","null"]},"persists_runs":{"enum":["all","errors_only","none"]},"attempts":{"type":["integer","null"]},"errors":{"type":["integer","null"]},"items":{"type":["integer","null"]},"last_run_at":{"type":["string","null"]},"last_success_at":{"type":["string","null"]},"last_error":{"type":["string","null"],"maxLength":300}}}}}}
{"$id":"schema://nova/job-health-monitor/read-filter@0.1.0","type":"object","properties":{"state":{"enum":["running","late","failing","no_work","on_demand","not_measured","off_on_purpose","unknown"]},"source":{"enum":["automation-hub","system-timers","assistant-scheduler","database-cron"]},"lane":{"enum":["scheduled","poller","on_demand"]}}}
{"$id":"schema://nova/job-health-monitor/summary@0.1.0","type":"object","required":["counts","stale_banner","badge","latest_cycle_at"],"properties":{"counts":{"type":"object","additionalProperties":{"type":"integer","minimum":0}},"stale_banner":{"type":"boolean"},"badge":{"enum":["green","red"]},"latest_cycle_at":{"type":["string","null"],"format":"date-time"}}}
{"$id":"schema://nova/job-health-monitor/alert@0.1.0","type":"object","required":["alert_id","job_id","from","to","reason","dedupe_key"],"properties":{"alert_id":{"type":"string"},"job_id":{"type":"string"},"from":{"type":"string"},"to":{"type":"string"},"reason":{"type":"string","maxLength":200},"dedupe_key":{"type":"string"},"sent_at":{"type":["string","null"],"format":"date-time"}}}
{"$id":"schema://nova/job-health-monitor/delivery@0.1.0","type":"object","required":["message_id","status"],"properties":{"message_id":{"type":"string"},"channel":{"type":"string"},"status":{"enum":["delivered","queued","failed"]},"delivered_at":{"type":["string","null"],"format":"date-time"}}}
```

**Example (fictional)**

Cycle output: `{"cycle_id":"cyc_20260504_0905","status":"completed","started_at":"2026-05-04T09:05:00Z","finished_at":"2026-05-04T09:05:02Z","sources":[{"source":"automation-hub","result":"ok","jobs_discovered":41},{"source":"system-timers","result":"ok","jobs_discovered":8},{"source":"assistant-scheduler","result":"ok","jobs_discovered":2}],"counts":{"running":38,"late":1,"failing":1,"no_work":2,"on_demand":6,"not_measured":3,"off_on_purpose":0},"transitions":[{"job_id":"hub:feed-refresh","from":"running","to":"failing"}],"alerts_sent":1}`

Row: `{"job_id":"hub:feed-refresh","source":"automation-hub","name":"feed-refresh","lane":"scheduled","enabled":true,"expected_interval_s":900,"next_run_at":null,"last_run_at":"2026-05-04T09:00:03Z","last_success_at":"2026-05-01T17:45:10Z","runs_window":96,"errors_window":96,"items_window":null,"state":"failing","display_state":"failing","state_reason":"96 attempts in 24 h, 0 succeeded","error_excerpt":"upstream returned HTTP 503 Service Unavailable","collected_at":"2026-05-04T09:05:02Z","present":true,"note":null}`

## E. Constraints and budgets

- **Hard constraints:** read-only access to every scheduler store (read-only open or private copy); never restart, re-run, enable, disable or edit a job; stale rows display `unknown`; `last_run_at` and `last_success_at` kept separate; `failing` only per R3; timers judged per R4; on-demand never `late`; one marker per outcome; all timestamps normalised to UTC; error excerpts redacted (tokens, keys, credentials in URLs) and ≤ 300 characters; only humans set `off_on_purpose`; alerts on state transitions only, never on a job's first observation.
- **Soft constraints:** keep the cycle under 10 s; group many simultaneous alerts into one summary.
- **Resource and tenant scope:** the operator's own estate and registry; one registry per tenant.
- **Data sources allowed:** the configured scheduler stores, journals and job files only.
- **External transfers permitted:** none beyond the owner's alert channel (alert text contains job names and short reasons, no payloads).
- **Time limit:** cycle ≤ 60 s; per source ≤ 20 s.
- **Cost or token budget:** no LLM calls in the cycle (deterministic rules); compute only.
- **Maximum steps:** 10 (S1–S10).
- **Maximum retries and correction cycles:** per source: 1 retry within the cycle; the next cycle is the natural retry; correction cycles: 1.
- **Rate limits and concurrency:** one cycle at a time (single-flight lock); ≤ 10 individual alerts per cycle, more become one summary.
- **Retention and logging constraints:** cycle records kept `TBD` (default 30 days); error excerpts only, never full execution payloads.
- **Prohibited actions:** writing to scheduler stores; opening the live hub store in read-write mode; reading full execution payloads into logs; greening a job without evidence.
- **Stop conditions:** another cycle holds the lock (skip); registry unavailable (cycle `failed`; rows go stale and display `unknown`).

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Job state row (newly discovered job) | discovered by a source variant | Allowed | row readable |
| Create | Cycle record; alert record | cycle lock held | Allowed | records readable |
| Read | Scheduler stores, journals, job files | read-only handles | Allowed | handle opened read-only |
| Update | Job row live columns (`state`, timestamps, counts, `error_excerpt`, `collected_at`, `present`) | collector identity; one transaction per cycle | Allowed | row `collected_at` = cycle time |
| Update | Job row human columns (`note`, `lane`, `off_on_purpose`) | human operator via `set` | Approval Required (human action) | audit entry |
| Delete | any | — | Not Allowed (a vanished job is marked `present: false`) | — |
| External side effect | state-change alert to the owner | transition detected; not first observation | Allowed | delivery receipt |

- **Idempotency:** the registry upsert is keyed by `job_id`; a cycle is keyed by `cycle_id` with a single-flight lock; alerts are keyed by `dedupe_key` = (`job_id`, `to`, transition cycle).
- **Expected side effects:** registry rows refreshed; occasional owner alerts.
- **Compensation:** a wrong state is corrected by the next cycle; a wrong alert is followed by a recovery message when the state returns.
- **Irreversible:** alerts already sent.
- **Ambiguous outcomes:** if the registry write times out, the next cycle re-reads and re-writes the full snapshot (idempotent upsert); alerts are sent only after the write commits.

## G. Autonomy and approval policy

- **Default autonomy level:** A3 — conditional autonomy for observation and reporting.
- **Planning level:** A3.
- **Execution level:** A3 (read-only collection; writes only to its own registry; alerts to the owner).
- **Verification level:** A3 with the cycle-level sanity challenge (S6).
- **Recovery level:** A3 within a cycle; persistent source failures escalate.
- **Approval required for:** changing thresholds or lanes; marking a job `off_on_purpose` (human action only).
- **Actions never permitted:** any action on a job itself (restart, re-run, enable, disable, edit); writing to a scheduler store; suppressing a `failing` state; sending alerts to anyone but the owner.
- **Conditions requiring clarification:** none at run time (deterministic rules).
- **Conditions requiring escalation:** a source failing for 3 consecutive cycles; the sanity challenge flags a source; the collector's own cycle record older than 20 minutes (external heartbeat).
- **Safe state after stop, cancellation or escalation:** registry unchanged by the interrupted cycle (single transaction); rows age into `unknown`; no job touched.

## H. Execution plan

**S1 — Start the cycle**
1. Objective: run exactly one cycle at a time.
2. Capability: `nova.jobs.registry.record_cycle` (open).
3. Input mapping: `trigger`, `now`, thresholds.
4. Dependencies: none.
5. Preconditions: schedule fired or manual request.
6. Action / side effects: take the single-flight lock; create the cycle record.
7. Autonomy: A3.
8. Timeout / retry: 5 s / none (skip if locked).
9. Idempotency: lock keyed by tenant; `cycle_id` from the start time.
10. Postconditions: lock held; cycle open.
11. Verification: lock owner = this cycle.
12. Failure routes: lock held by a live cycle → skip (`cancelled`); stale lock older than 10 minutes → take over and record it.
13. Escalation triggers: none.
14. Evidence: `cycle_id`, lock record.

**S2 — Discover jobs**
1. Objective: the complete current job list from each source.
2. Capability: `nova.jobs.sources.discover` (`.automation-hub`, `.system-timers`, `.assistant-scheduler`, `.database-cron`).
3. Input mapping: configured sources.
4. Dependencies: S1.
5. Preconditions: read-only handles available.
6. Action: list jobs with trigger type (schedule, poller, webhook or event), enabled flag, expected interval.
7. Autonomy: A3.
8. Timeout / retry: 20 s per source / 1 retry.
9. Idempotency: read-only.
10. Postconditions: per-source job lists, or a source error.
11. Verification: handle mode is read-only; list non-empty unless the source is genuinely empty.
12. Failure routes: source error → mark the source `error`; its rows are not refreshed (they turn `unknown` at 20 minutes).
13. Escalation: none here (see S6, S9).
14. Evidence: counts per source, handle mode.

**S3 — Collect run evidence**
1. Objective: the facts needed by the rules.
2. Capability: `nova.jobs.sources.collect_runs` (same variants).
3. Input mapping: job IDs per source; window = failure window (or longer per R3).
4. Dependencies: S2.
5. Preconditions: source `ok`.
6. Action: read last run, last success, attempts, errors, items (if exposed), next-run time, sub-state, last error message (decoded, not regex-matched).
7. Autonomy: A3.
8. Timeout / retry: 20 s per source / 1 retry.
9. Idempotency: read-only.
10. Postconditions: evidence per job.
11. Verification: queries bounded by the window; no full payloads read into memory beyond the error field.
12. Failure routes: partial failure → affected jobs keep their previous rows (stale later).
13. Escalation: none.
14. Evidence: query timings, rows read.

**S4 — Normalise**
1. Objective: comparable, de-duplicated evidence.
2. Capability: part of `nova.jobs.evaluate` (normalisation stage).
3. Input mapping: raw evidence.
4. Dependencies: S3.
5. Preconditions: evidence present.
6. Action: convert all timestamps to UTC (naive-UTC, offset-aware, formatted and epoch inputs); keep one canonical marker per outcome (R7); assign lanes (R9); redact and truncate error excerpts.
7. Autonomy: A3.
8. Timeout / retry: 5 s / none.
9. Idempotency: pure transformation.
10. Postconditions: normalised evidence.
11. Verification: no naive timestamps remain; marker counts ≤ raw line counts.
12. Failure routes: unparseable timestamp → that job gets `unknown` reason "unparseable time" (stored state left unchanged, flagged).
13. Escalation: none.
14. Evidence: normalisation warnings.

**S5 — Derive states**
1. Objective: one honest state and reason per job.
2. Capability: `nova.jobs.evaluate.derive_state`.
3. Input mapping: normalised evidence, thresholds, human columns.
4. Dependencies: S4.
5. Preconditions: evidence normalised.
6. Action: apply precedence `off_on_purpose` → `failing` (R3) → `late` (R4, R5) → `not_measured` (R6) → `on_demand` → `no_work` → `running`; write a one-line reason with the numbers.
7. Autonomy: A3.
8. Timeout / retry: 5 s / none.
9. Idempotency: pure function.
10. Postconditions: state per job.
11. Verification: every `running` job has `last_success_at` inside its window or an executing service; no on-demand job is `late`.
12. Failure routes: rule error for one job → that job keeps its previous row, flagged.
13. Escalation: none.
14. Evidence: state and reason per job.

**S6 — Sanity challenge (cycle level)**
1. Objective: catch collector errors before they become false alarms or false greens.
2. Capability: rule checks within `nova.jobs.evaluate`.
3. Input mapping: this cycle's results vs the previous cycle.
4. Dependencies: S5.
5. Preconditions: states derived.
6. Action: flag a source `suspect` if its job count drops by more than 30 % or to zero, or if more than 80 % of its scheduled jobs turn `late` at once (likely a time-parsing or clock problem); suspect sources are not written.
7. Autonomy: A3.
8. Timeout / retry: 2 s / none.
9. Idempotency: pure.
10. Postconditions: sources `ok` or `suspect`.
11. Verification: suspect sources recorded with the reason.
12. Failure routes: suspect → keep previous rows (they age into `unknown`), raise one operator alert instead of many job alerts.
13. Escalation: suspect for 3 cycles → escalate.
14. Evidence: challenge results.

**S7 — Write the registry**
1. Objective: persist the snapshot atomically.
2. Capability: `nova.jobs.registry.report`.
3. Input mapping: job rows from `ok` sources; `present: false` for jobs missing from an `ok` source.
4. Dependencies: S6.
5. Preconditions: not `dry_run`.
6. Action / side effects: upsert all rows in one transaction with `collected_at` = cycle time; human columns untouched.
7. Autonomy: A3.
8. Timeout / retry: 10 s / 1 retry.
9. Idempotency: upsert by `job_id`.
10. Postconditions: rows refreshed.
11. Verification: rows written = rows computed; `collected_at` equal on all.
12. Failure routes: write fails → roll back; cycle `failed`; rows age into `unknown`.
13. Escalation: 3 consecutive failed writes → escalate.
14. Evidence: rows written.

**S8 — Detect transitions and alert**
1. Objective: tell the owner about real changes only.
2. Capability: `nova.notify.owner.send`.
3. Input mapping: previous vs new state per job.
4. Dependencies: S7 committed.
5. Preconditions: not `dry_run`.
6. Action / side effects: alert on healthy (`running`, `no_work`, `on_demand`) → `failing` or `late`, and on `failing` or `late` → `running`; never on a first observation; more than 10 alerts → one summary.
7. Autonomy: A3.
8. Timeout / retry: 15 s / 2 retries.
9. Idempotency: `dedupe_key`.
10. Postconditions: alerts sent or queued for the next cycle.
11. Verification: delivery receipt per alert.
12. Failure routes: channel down → keep the alert pending and retry next cycle (send-then-mark).
13. Escalation: none.
14. Evidence: alert records.

**S9 — Close the cycle**
1. Objective: an auditable cycle record.
2. Capability: `nova.jobs.registry.record_cycle` (close).
3. Input mapping: source results, counts, transitions, alerts.
4. Dependencies: S8.
5. Preconditions: cycle open.
6. Action / side effects: write the cycle result; release the lock.
7. Autonomy: A3.
8. Timeout / retry: 5 s / 1 retry.
9. Idempotency: by `cycle_id`.
10. Postconditions: cycle closed; lock released.
11. Verification: counts sum to rows written.
12. Failure routes: close fails → lock expires after 10 minutes.
13. Escalation: a source in `error` for 3 cycles → operator alert.
14. Evidence: `cycle_result`.

**S10 — Serve reads with the staleness rule**
1. Objective: every reader sees honest states.
2. Capability: `nova.jobs.registry.list` / `.summary`.
3. Input mapping: `read_filter`.
4. Dependencies: registry rows (any cycle).
5. Preconditions: reader authorised.
6. Action: compute `display_state` = `unknown` when `collected_at` is older than 20 minutes; set the stale banner when the latest cycle is older than 20 minutes; badge red if any `failing`, `late` or `unknown`.
7. Autonomy: A3.
8. Timeout / retry: 5 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: list or summary returned.
11. Verification: no row older than 20 minutes displays anything but `unknown`.
12. Failure routes: registry unreadable → return an error, never an empty "all good".
13. Escalation: none.
14. Evidence: read timestamp.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Read-only access | every source handle opened read-only or on a copy | S2 handle log | all read-only | abort that source; escalate |
| Coverage | every enabled job in each `ok` source appears in the registry | S2 vs S7 | 100 % | flag missing jobs |
| Staleness | rows older than 20 min display `unknown` | S10 | always | fix read path; treat as defect |
| Success evidence | each `running` job has a recent success or an executing service | S5 | all | downgrade to `unknown` |
| Failing threshold | `failing` only per R3 | S5 reason numbers | all | re-derive |
| Timer judgement | timer states use next-run time and sub-state | S5 reason | all | re-derive |
| Marker counting | runs counted once per outcome | S4 | markers ≤ runs | fix parser |
| Time normalisation | no naive timestamps | S4 | none | `unknown` for affected jobs |
| Alert discipline | alerts only on transitions, none on first observation | S8 | pass | suppress and log |
| Output schema | `cycle_result` and rows valid | validator | valid | fix fields; never fake values |

A source query returning rows proves only that the store was readable, not that a job is healthy.

## J. Self-correction and recovery

- **Failure classes:** lock contention, source unavailable, permission denied, slow source (timeout), unparseable data, suspect source, registry write failure, alert delivery failure.
- **Permitted corrections:** one retry per source per cycle; take over a stale lock; queue undelivered alerts for the next cycle; let the next cycle refresh rows.
- **Forbidden corrections:** marking jobs `running` without evidence; widening thresholds to silence alerts; writing to scheduler stores; dropping vanished jobs instead of marking `present: false`; hiding a stale banner.
- **Maximum correction cycles:** 1 per source per cycle.
- **Reverification required after correction:** yes — the S6 sanity challenge runs on every cycle.
- **Stop conditions:** registry unavailable; lock held by a live cycle.
- **Recovery or compensation:** the next successful cycle overwrites the snapshot; a recovery alert follows a resolved failure.
- **What must be reported:** sources in error or suspect, rows written, transitions, alerts sent or pending.

## K. Independent challenge

- **Assumptions to challenge:** the source shows all jobs; a success record means the job did useful work; the scheduler's next-run time is truthful.
- **Alternative approach:** for a critical job, would an end-to-end probe (checking its output, for example the newest synced record's age) be a better signal than run records?
- **Contradictory evidence:** a job `running` while its downstream output has not changed for days (surfaced as `no_work` where item counts exist).
- **Could mislead:** a mass `late` caused by a time-zone parsing error; a green board while the collector itself has stopped.
- **Separate verifier:** the S6 sanity challenge compares cycles; the read path enforces staleness independently of the collector; an external heartbeat for the collector.
- **Findings requiring disclosure:** not-measured jobs, suspect sources, jobs flagged as running while `off_on_purpose`.

## L. Human escalation

- **Triggers:** a source in `error` or `suspect` for 3 consecutive cycles; registry writes failing for 3 cycles; collector stale for more than 20 minutes (external heartbeat); a job flagged as running while `off_on_purpose`.
- **Why the agent cannot decide:** fixing jobs or access is outside its read-only scope.
- **Information presented:** source, error or suspect reason, affected job count, last good cycle, example rows.
- **Decision requested:** repair access or the source; confirm a mass change is real; update thresholds or lanes.
- **Options and consequences:** for example, ignoring a suspect source leaves its jobs `unknown`.
- **Safe behaviour while waiting:** keep cycling; affected rows stay `unknown`; no job touched.
- **If no response:** escalation repeats daily, not every cycle.
- **Resume:** automatic on the next clean cycle.

## M. Completion conditions

Mandatory: `cycle_result` valid; every `ok` source's jobs written in one transaction; every state derived by the rules with a reason; staleness enforced at read time; alerts only on transitions; limitations disclosed; evidence references present.

- `completed` — all configured sources `ok`, registry written, alerts handled.
- `partially_completed` — at least one source `error` or `suspect`, the rest written; or alerts pending delivery.
- `blocked` — registry unreachable or required read access denied for every source.
- `failed` — the registry write failed and was rolled back.
- `cancelled` — skipped because another cycle held the lock, or stopped by the operator.

## N. Final output contract

Return: run (cycle) ID; recipe ID and version; status; one-line outcome (for example "51 jobs: 38 running, 1 failing, 1 late"); `cycle_result`; step statuses S1–S10; verification results and failed checks; evidence references (cycle record, handle modes, rows written, alert IDs); approvals and scope (none at run time); side effects (registry writes, alerts) and reconciliation status; corrections; unresolved challenges (suspect sources); limitations (not-measured jobs); escalation state; learning candidate. Never include credentials, full execution payloads or unredacted error text.

## O. Learning and revision

- **Outcome metrics:** time from a real outage to `failing`, false-alarm rate (alerts followed by recovery within one cycle), share of `not_measured` and `unknown`, cycle duration, source error rate.
- **Evidence retained:** cycle records and alert records.
- **Data minimisation:** error excerpts redacted and truncated; no execution payloads.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}` — each new false-state class becomes a rule plus a regression test.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review; a revision can never weaken R1 (stale ≠ healthy) or grant write access to scheduler stores.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (fixture estate across three sources) | every job discovered; states match expectations; one cycle record | UNVERIFIED on Skill Port; observed in the reference implementation |
| Invalid input (unknown source, out-of-range threshold) | ignored or defaulted with a warning | TBD |
| Missing input (no thresholds) | defaults applied | TBD |
| Missing capability (assistant source binding absent) | source `error`; its rows age into `unknown` | TBD |
| Permission denied (hub store unreadable) | source `error`; no greening | TBD |
| Unauthorized scope (attempt to open a store read-write) | refused; source aborted | TBD |
| Tenant isolation | a reader of tenant A cannot see tenant B's registry | TBD |
| Timeout (slow source) | one retry, then `error`; other sources unaffected | TBD |
| Duplicate / replay (two cycles start together) | single-flight: one runs, one `cancelled` | TBD |
| Double log markers per run | counted once | TBD |
| Rule: blip (1 error in 1,800 runs) | `running` | TBD |
| Rule: 4 errors in 10 runs vs 3 errors in 10 runs | `failing` vs `running` | TBD |
| Rule: timer outside its window, next run tomorrow | `running` | TBD |
| Rule: timer service executing now (next-run blank) | `running` | TBD |
| Rule: on-demand job silent for a week | `on_demand`, never `late` | TBD |
| Rule: scheduler keeps no run records | `not_measured` | TBD |
| Rule: naive vs offset timestamps for the same instant | equal after normalisation | TBD |
| False success (collector stopped 30 minutes) | all rows display `unknown`; stale banner | TBD |
| Failed verification (sanity challenge: job count drops 50 %) | source `suspect`; previous rows kept; one operator alert | TBD |
| Correction budget exhausted (source fails 3 cycles) | escalation | TBD |
| Escalation and alert discipline (first observation; transition; recovery) | no alert; one alert; one recovery alert | TBD |
| Cancellation (operator stops a cycle) | no partial write | TBD |
| Rollback (registry write fails mid-transaction) | nothing committed; cycle `failed` | TBD |
| Output schema and status | valid `cycle_result` and rows | TBD |
| Compatibility / regression (scheduler version prints formatted times; calendar normalisation changes) | next-run parsed; states unchanged | TBD |

All tests run against fixture stores and a sandbox registry; never against a production scheduler with write access. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, schema registry and how read-only access to host schedulers is granted — TBD (owner: publisher, with Skill Port).
- Elapsed-time tolerance for sources without a next-run time and the failure window for long-cadence jobs — TBD (owner: operator).
- The external heartbeat for the collector itself needs a scheduler independent of the collector (owner: operator).
- `no_work` depends on sources exposing item counts; many do not.
- Jobs whose scheduler discards successful runs can only be judged on errors.
- Retention of cycle and alert records — TBD (owner: operator).
