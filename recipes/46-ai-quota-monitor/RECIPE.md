# AI quota monitor

## A. Identity and ownership

- **Recipe ID:** `nova.ops.ai-quota-monitor` (proposed until assigned by Skill Port)
- **Name:** AI quota monitor
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** operations
- **Tags:** ai-quota, usage-window, credit-balance, stale-detection, threshold-alerts, fallback-check
- **Summary:** Every 30 minutes the agent snapshots each AI subscription's usage window (percent used, window length, reset time) and each pay-as-you-go provider's credit balance, keeps the last good snapshot when a fetch fails, shows any snapshot older than 3 hours as `stale` rather than "0 % used", and alerts the owner once when a window crosses 75 % (warning) or 90 % (critical) or a balance runs low or dry.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** A read-only way to obtain each subscription's usage window (the assistant runtime's own account-usage function for the providers it supports, or a provider's documented usage endpoint); a credit-balance source for pay-as-you-go providers (provider balance endpoint, or the usage ledger from `nova.ops.cost-per-job` plus operator-recorded purchases); a provider registry with policy text; a snapshot store; a scheduler (every 30 minutes); an owner notification channel. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — the core flow (30-minute window snapshots stored alongside policy text, with stale flagging) runs in Nova's reference implementation; not every control described here has evidence yet (Section P lists the tests still needed). Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.quota.providers` | Capability | Registry of AI backends with role, kind, policy text and fallback chain | low | UNVERIFIED |
| `nova.quota.providers.list` | Operation (read) | Return every configured backend | low | UNVERIFIED |
| `nova.quota.windows` | Capability | Usage-window snapshots for subscriptions | medium | UNVERIFIED |
| `nova.quota.windows.fetch` | Operation (read) | Percent used, window length and reset time per window | medium | UNVERIFIED |
| `nova.quota.balance` | Capability | Credit balances for pay-as-you-go providers | medium | UNVERIFIED |
| `nova.quota.balance.fetch` | Operation (read) | Balance, currency, basis and runway | medium | UNVERIFIED |
| `nova.quota.signals` | Capability | Evidence that a provider has run dry | low | UNVERIFIED |
| `nova.quota.signals.scan` | Operation (read) | Recent exhaustion errors (for example insufficient-credit responses) per provider | low | UNVERIFIED |
| `nova.quota.store` | Capability | Snapshot store and read views | low | UNVERIFIED |
| `nova.quota.store.upsert_snapshot` | Operation (create/update) | Write live columns only; keep `last_good`; never touch policy text | low | UNVERIFIED |
| `nova.quota.store.record_cycle` | Operation (create/update) | Open and close a cycle record with a single-flight lock | low | UNVERIFIED |
| `nova.quota.store.list` | Operation (read) | Display view with the 3-hour stale rule applied | low | UNVERIFIED |
| `nova.quota.evaluate` | Capability | Thresholds, runway and fallback independence | low | UNVERIFIED |
| `nova.quota.evaluate.assess` | Operation (pure function) | Level per window and balance; chain verdicts | low | UNVERIFIED |
| `nova.notify.owner` | Capability | Deliver a message to the owner's channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Threshold, exhaustion and recovery notices | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.quota.windows.fetch.runtime-account-usage` — calls the assistant runtime's own account-usage function for the providers it supports. It must be side-effect-free: the collector hashes the runtime's provider configuration before and after each call, and a changed hash stops the variant and raises a critical operator alert (the hash proves the read had no side effect on configuration). Binding `binding://nova/quota/runtime-account-usage`.
- `nova.quota.windows.fetch.provider-usage-api` — a provider's documented usage endpoint, read-only. Binding `binding://nova/quota/provider-usage-api`.
- Window variants are selected per provider by configuration (whichever the provider supports); a provider supported by neither is `not_measured` — never `0 %`.
- `nova.quota.balance.fetch.provider-balance-api` — the provider's balance endpoint, read-only. Selected first. Binding `binding://nova/quota/provider-balance-api`.
- `nova.quota.balance.fetch.ledger-derived` — operator-recorded purchased credits minus spend recorded in the usage ledger since the purchase; an estimate, labelled as such. Eligible only when the provider has no balance endpoint; read-only like the first, same scopes, disclosed. Binding `binding://nova/quota/ledger-derived`.
- `nova.quota.signals.scan.default` — always runs in addition to the balance variants; a recent exhaustion error overrides any balance reading to `exhausted`.
- Every other operation supports only the default variant.

**Levels:**

| Subject | `ok` | `warn` | `critical` | `exhausted` | Other |
|---|---|---|---|---|---|
| Subscription window | < 75 % used | ≥ 75 % | ≥ 90 % | ≥ 100 % or limit reported | `not_measured` (no variant supports it) |
| Credit balance | runway ≥ 7 days and above the warning floor | runway < 7 days or below the warning floor | runway < 2 days or below the critical floor | balance ≤ 0, or an exhaustion signal in the last 60 minutes | `not_measured` (no balance source) |
| Any snapshot (display) | — | — | — | — | `stale` when older than 3 hours |

Runway = balance ÷ average daily spend over the last 7 days. The floors and day counts are operator settings (`TBD`; 7 and 2 days are illustrative defaults).

**Monitor rules:**

- **Q1 Stale ≠ empty:** a snapshot older than 3 hours displays `stale` with its age; the last known value may be shown only with its timestamp and never drives an `ok` colour. A missing value is `null`, never `0`.
- **Q2 Last good kept:** a failed fetch records `last_fetch_error` and leaves the last good snapshot in place; it never overwrites it with zeros.
- **Q3 Policy text is configuration:** each backend's role and policy text are owned by configuration; snapshot writes touch live columns only, so a failed fetch can never erase the policy.
- **Q4 Worst window wins:** subscriptions with several windows (for example a short rolling window and a weekly window) are snapshotted per window; the backend's level is the worst window.
- **Q5 Alert once per crossing:** one alert per (backend, window, reset time, level), including when the first snapshot is already above a threshold; a recovery notice once when a reset or top-up brings it back to `ok`.
- **Q6 Independent fallback:** a primary backend whose fallback chain uses only the same billing account has no independent fallback → `warn`; a primary at `critical`, `exhausted` or `stale` whose fallbacks are all `exhausted`, `stale` or `not_measured` → `critical` "no working backend".
- **Q7 Observe only:** the monitor never changes routing, switches models, buys or tops up credits, or edits runtime configuration.

**Bindings:** the five variant bindings above plus `binding://nova/quota/providers`, `binding://nova/quota/signals`, `binding://nova/quota/store`, `binding://nova/quota/evaluate`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: a collector on a 30-minute system timer calling the assistant runtime's account-usage function and writing through a usage dispatcher (a read action and a collector-only upsert action) into a table that holds both policy text and live quota, shown on a costs page with stale flags after 3 hours.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Provider entry | `schema://nova/ai-quota-monitor/provider@0.1.0` | operator configuration | read |
| Window snapshot | `schema://nova/ai-quota-monitor/window-snapshot@0.1.0` | the operator's tenant | create, update (live columns) |
| Balance snapshot | `schema://nova/ai-quota-monitor/balance-snapshot@0.1.0` | the operator's tenant | create, update (live columns) |
| Exhaustion signal | `schema://nova/ai-quota-monitor/error-signal@0.1.0` | derived | read |
| Cycle record | `schema://nova/ai-quota-monitor/cycle-result@0.1.0` | the operator's tenant | create, update |
| Alert | `schema://nova/ai-quota-monitor/quota-alert@0.1.0` | the owner | create |

**Authorization requirements (declared, never a grant):** `quota:providers:read`, `quota:usage:read` (per provider, read-only account or usage scope), `quota:balance:read`, `usage:ledger:read` (ledger-derived variant), `runtime:account_usage:invoke` (read-only invocation), `quota:store:write` (collector identity), `quota:store:read`, `notify:owner`. Provider credentials stay in the runtime as `secret_ref://<provider>-account`; the collector never receives billing-write or purchase permissions.

**Policy / trust:** risk class `medium` (account-level read access; misleading quota status could leave the assistant without a working model). Observe-only; config-hash guard on runtime calls.

## C. Intent and applicability

- **User problem:** The assistant depends on subscriptions with usage windows and on prepaid pay-as-you-go balances; when one runs out, scheduled jobs and the assistant itself can stop working without warning.
- **Desired outcome:** A current, honest picture of every AI backend's headroom, and one clear alert before (not after) a window or balance runs out.
- **Trigger or entry conditions:** The 30-minute schedule; a manual refresh; a read of the quota view.
- **Applicable situations:** Subscription plans with rolling or weekly windows; prepaid credit balances; free tiers (shown as `not_measured` unless a source exists).
- **Out-of-scope situations:** Changing routing or models; buying credits; per-job cost attribution (see `nova.ops.cost-per-job`); job failures in general (see `nova.ops.job-health-monitor`).
- **Required inputs:** none for scheduled cycles (Section D lists optional inputs).
- **Prerequisites:** Provider registry configured with roles, billing accounts and fallback chains; read-only usage and balance access; thresholds set or defaults accepted.
- **Required capabilities:** `nova.quota.providers`, `nova.quota.windows`, `nova.quota.balance`, `nova.quota.signals`, `nova.quota.store`, `nova.quota.evaluate`, `nova.notify.owner` (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port. Many providers expose no balance endpoint; then only the ledger-derived estimate and error signals are available.
- **Assumptions:** The core flow runs in the reference implementation. Rationale for balance monitoring: prepaid balances can run out without a provider-side alert, so the monitor watches them directly. Assumed: purchase records are entered by the operator for the ledger-derived variant.
- **Ambiguity policy:** Use documented safe defaults: unknown → `not_measured`; old → `stale`; contradictory (balance positive but recent exhaustion errors) → `exhausted` with both facts shown.
- **Vedaxi task type:** none — this recipe is not one of the seven task types Vedaxi supports today.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `trigger` | enum `schedule`, `manual` | yes | scheduler / operator | `schedule` | Invalid → reject the cycle. |
| `providers` | array of provider IDs | no | operator | all configured | Unknown IDs ignored with a warning. |
| `thresholds` | object (below) | no | operator config | warn 75, critical 90, stale 3 h, runway 7 / 2 days | Out of range → defaults with a warning. |
| `now` | date-time | no | test harness only | system clock | Only honoured in test mode. |
| `dry_run` | boolean | no | operator / tests | `false` | `true` → compute only. |

```json
{"$id":"schema://nova/ai-quota-monitor/cycle-request@0.1.0","type":"object","required":["trigger"],"properties":{"trigger":{"enum":["schedule","manual"]},"providers":{"type":"array","items":{"type":"string"},"uniqueItems":true},"thresholds":{"type":"object","properties":{"warn_pct":{"type":"number","minimum":50,"maximum":95,"default":75},"critical_pct":{"type":"number","minimum":60,"maximum":99,"default":90},"stale_after_h":{"type":"number","minimum":1,"maximum":24,"default":3},"runway_warn_days":{"type":"number","minimum":1,"default":7},"runway_critical_days":{"type":"number","minimum":0.5,"default":2},"warn_floor":{"type":["number","null"],"minimum":0},"critical_floor":{"type":["number","null"],"minimum":0}}},"now":{"type":"string","format":"date-time"},"dry_run":{"type":"boolean"}}}
{"$id":"schema://nova/ai-quota-monitor/provider@0.1.0","type":"object","required":["provider_id","kind","role","billing_account","policy_text"],"properties":{"provider_id":{"type":"string"},"display_name":{"type":"string"},"kind":{"enum":["subscription","pay_as_you_go","free_tier"]},"role":{"enum":["primary","fallback","bulk","automation"]},"billing_account":{"type":"string"},"policy_text":{"type":"string","maxLength":500},"window_variant":{"enum":["runtime-account-usage","provider-usage-api",null]},"balance_variant":{"enum":["provider-balance-api","ledger-derived",null]},"fallback_chain":{"type":"array","items":{"type":"string"}}}}
{"$id":"schema://nova/ai-quota-monitor/window-snapshot@0.1.0","type":"object","required":["provider_id","window","fetched_at"],"properties":{"provider_id":{"type":"string"},"window":{"type":"string"},"window_length_h":{"type":["number","null"],"minimum":0},"pct_used":{"type":["number","null"],"minimum":0,"maximum":100},"resets_at":{"type":["string","null"],"format":"date-time"},"fetched_at":{"type":"string","format":"date-time"},"variant":{"enum":["runtime-account-usage","provider-usage-api"]},"last_fetch_error":{"type":["string","null"],"maxLength":200},"config_hash_unchanged":{"type":["boolean","null"]}}}
{"$id":"schema://nova/ai-quota-monitor/balance-snapshot@0.1.0","type":"object","required":["provider_id","basis","fetched_at"],"properties":{"provider_id":{"type":"string"},"balance":{"type":["number","null"]},"currency":{"type":["string","null"]},"basis":{"enum":["provider_reported","ledger_estimate"]},"avg_daily_spend_7d":{"type":["number","null"],"minimum":0},"runway_days":{"type":["number","null"],"minimum":0},"fetched_at":{"type":"string","format":"date-time"},"last_fetch_error":{"type":["string","null"],"maxLength":200}}}
{"$id":"schema://nova/ai-quota-monitor/error-signal@0.1.0","type":"object","required":["provider_id","signal","observed_at","source"],"properties":{"provider_id":{"type":"string"},"signal":{"enum":["insufficient_credit","quota_exceeded","rate_limited","payment_required"]},"observed_at":{"type":"string","format":"date-time"},"source":{"type":"string"},"count_60m":{"type":"integer","minimum":1}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `cycle_result` | object `cycle-result@0.1.0` | `cycle_id`, `status`, `providers` | cycle record; run record | internal account data | snapshots re-read after write (Section I) |
| `quota_view` | array of `quota-view@0.1.0` | per schema | costs or quota page, agent answers | internal | stale rule applied at read time |
| `quota_alert` | object `quota-alert@0.1.0` | per schema | owner channel | internal | deduplicated per crossing |

```json
{"$id":"schema://nova/ai-quota-monitor/evaluation@0.1.0","type":"object","required":["provider_id","level","reasons"],"properties":{"provider_id":{"type":"string"},"level":{"enum":["ok","warn","critical","exhausted","not_measured"]},"worst_window":{"type":["string","null"]},"reasons":{"type":"array","items":{"type":"string","maxLength":160}},"independent_fallback":{"type":["boolean","null"]},"no_working_backend":{"type":"boolean"}}}
{"$id":"schema://nova/ai-quota-monitor/quota-alert@0.1.0","type":"object","required":["alert_id","provider_id","level","dedupe_key","message"],"properties":{"alert_id":{"type":"string"},"provider_id":{"type":"string"},"window":{"type":["string","null"]},"level":{"enum":["warn","critical","exhausted","recovered","no_independent_fallback","no_working_backend","config_changed"]},"dedupe_key":{"type":"string"},"message":{"type":"string","maxLength":300},"sent_at":{"type":["string","null"],"format":"date-time"}}}
{"$id":"schema://nova/ai-quota-monitor/cycle-result@0.1.0","type":"object","required":["cycle_id","status","providers"],"properties":{"cycle_id":{"type":"string"},"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},"started_at":{"type":"string","format":"date-time"},"finished_at":{"type":"string","format":"date-time"},"providers":{"type":"array","items":{"type":"object","required":["provider_id","fetch"],"properties":{"provider_id":{"type":"string"},"fetch":{"enum":["ok","error","not_measured","skipped"]},"level":{"type":["string","null"]}}}},"alerts_sent":{"type":"integer","minimum":0}}}
{"$id":"schema://nova/ai-quota-monitor/quota-view@0.1.0","type":"object","required":["provider_id","role","policy_text","display_state"],"properties":{"provider_id":{"type":"string"},"role":{"type":"string"},"policy_text":{"type":"string"},"display_state":{"enum":["ok","warn","critical","exhausted","not_measured","stale"]},"windows":{"type":"array","items":{"type":"object","properties":{"window":{"type":"string"},"pct_used":{"type":["number","null"]},"resets_at":{"type":["string","null"]},"age_minutes":{"type":"integer"}}}},"balance":{"type":["number","null"]},"balance_basis":{"type":["string","null"]},"runway_days":{"type":["number","null"]},"last_good_at":{"type":["string","null"],"format":"date-time"}}}
{"$id":"schema://nova/ai-quota-monitor/delivery@0.1.0","type":"object","required":["message_id","status"],"properties":{"message_id":{"type":"string"},"channel":{"type":"string"},"status":{"enum":["delivered","queued","failed"]},"delivered_at":{"type":["string","null"],"format":"date-time"}}}
```

**Example (fictional)**

Cycle output: `{"cycle_id":"qta_20260504_0930","status":"completed","started_at":"2026-05-04T09:30:00Z","finished_at":"2026-05-04T09:30:04Z","providers":[{"provider_id":"assistant-plan","fetch":"ok","level":"warn"},{"provider_id":"payg-gateway","fetch":"ok","level":"critical"},{"provider_id":"automation-llm","fetch":"not_measured","level":"not_measured"}],"alerts_sent":2}`

View row: `{"provider_id":"payg-gateway","role":"bulk","policy_text":"Pay-as-you-go backend for background batch jobs.","display_state":"critical","windows":[],"balance":1.40,"balance_basis":"provider_reported","runway_days":1.6,"last_good_at":"2026-05-04T09:30:03Z"}` (amounts fictional).

## E. Constraints and budgets

- **Hard constraints:** observe only (Q7); read-only usage and balance access; runtime account-usage calls must leave the runtime configuration hash unchanged; stale after 3 hours displays `stale`, never `0 %` (Q1); failed fetches keep the last good snapshot (Q2); snapshot writes never touch policy text (Q3); missing values are `null`; alerts once per crossing (Q5); a recent exhaustion signal overrides a positive balance.
- **Soft constraints:** finish each cycle in under 10 seconds; send `warn` alerts outside the owner's quiet hours (`critical` and `exhausted` always go through).
- **Resource and tenant scope:** the operator's configured backends and snapshot store.
- **Data sources allowed:** the configured usage and balance sources, the usage ledger, recent call errors from the operator's own logs or job records.
- **External transfers permitted:** none beyond owner alerts (backend names, percentages, balances, reset times).
- **Time limit:** cycle ≤ 60 s; per provider fetch ≤ 15 s.
- **Cost or token budget:** no LLM calls; usage reads must not consume billable quota where the provider distinguishes them (`TBD` per provider).
- **Maximum steps:** 11 (S1–S11).
- **Maximum retries and correction cycles:** per provider: 1 retry per cycle; the next cycle is the natural retry.
- **Rate limits and concurrency:** one cycle at a time; provider read limits apply.
- **Retention and logging constraints:** snapshot history kept `TBD` (default 30 days); logs never contain credentials or account identifiers beyond the provider ID.
- **Prohibited actions:** switching models or routing; purchasing or topping up; editing runtime configuration; writing zeros for unknown values; overwriting policy text.
- **Stop conditions:** another cycle holds the lock (skip); store unavailable (cycle `failed`; view shows `stale` after 3 hours); config hash changed (stop that variant, critical alert).

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Window and balance snapshots (first time per provider/window) | provider configured | Allowed | snapshot readable |
| Create | Cycle record; alert record | lock held | Allowed | records readable |
| Read | Provider registry, usage and balance sources, ledger, recent errors | read-only | Allowed | — |
| Update | Snapshot live columns (`pct_used`, `resets_at`, `balance`, `runway_days`, `fetched_at`, `last_fetch_error`) | collector identity | Allowed | re-read equals written values |
| Update | Provider registry, policy text, thresholds | operator configuration change | Approval Required (human, outside the cycle) | audit entry |
| Delete | any | — | Not Allowed | — |
| External side effect | alerts to the owner | crossing detected; deduplicated | Allowed | delivery receipt |

- **Idempotency:** snapshot upsert keyed by (`provider_id`, `window`); cycles keyed by `cycle_id` with a single-flight lock; alerts keyed by `dedupe_key` = (provider, window, reset time, level).
- **Expected side effects:** snapshot rows refreshed; occasional alerts.
- **Compensation:** a wrong level is corrected on the next cycle; a recovery notice follows a resolved condition.
- **Irreversible:** alerts already sent.
- **Ambiguous outcomes:** if a snapshot write times out, re-read; the next cycle rewrites the full set (idempotent). A runtime call whose config hash check is inconclusive is treated as changed.

## G. Autonomy and approval policy

- **Default autonomy level:** A3 — conditional autonomy for observation and alerts.
- **Planning level:** A3.
- **Execution level:** A3 (read-only fetches; writes only to its own store; owner alerts).
- **Verification level:** A3.
- **Recovery level:** A3 within a cycle; persistent failures escalate.
- **Approval required for:** changing thresholds, roles, fallback chains or policy text (human configuration change).
- **Actions never permitted:** changing routing, models or runtime configuration; buying credits; suppressing a `critical` or `exhausted` alert; displaying `0 %` for unknown data.
- **Conditions requiring clarification:** none at run time (deterministic rules).
- **Conditions requiring escalation:** `critical`, `exhausted`, "no working backend", "no independent fallback", config hash changed, a provider fetch failing for 6 consecutive cycles (3 hours).
- **Safe state after stop, cancellation or escalation:** last good snapshots retained; view turns `stale` after 3 hours; nothing about routing changed.

## H. Execution plan

**S1 — Start the cycle**
1. Objective: one cycle at a time.
2. Capability: `nova.quota.store.record_cycle` (open).
3. Input mapping: `cycle-request@0.1.0`.
4. Dependencies: none.
5. Preconditions: schedule fired or manual request.
6. Action / side effects: take the lock; open the cycle record.
7. Autonomy: A3.
8. Timeout / retry: 5 s / none (skip if locked).
9. Idempotency: lock per tenant; `cycle_id` from the start time.
10. Postconditions: lock held.
11. Verification: lock owner = this cycle.
12. Failure routes: locked by a live cycle → `cancelled`; stale lock older than 30 minutes → take over and record.
13. Escalation triggers: none.
14. Evidence: `cycle_id`.

**S2 — Load the provider registry**
1. Objective: know which backends to check and how.
2. Capability: `nova.quota.providers.list`.
3. Input mapping: `providers` filter.
4. Dependencies: S1.
5. Preconditions: registry readable.
6. Action: read kind, role, billing account, variants, fallback chain, policy text.
7. Autonomy: A3.
8. Timeout / retry: 5 s / 1 retry.
9. Idempotency: read-only.
10. Postconditions: provider list.
11. Verification: every provider has a role and billing account.
12. Failure routes: registry unreadable → `blocked`.
13. Escalation: none.
14. Evidence: provider count.

**S3 — Snapshot subscription windows**
1. Objective: percent used and reset time per window.
2. Capability: `nova.quota.windows.fetch` (`runtime-account-usage` or `provider-usage-api`).
3. Input mapping: subscription providers and their variants.
4. Dependencies: S2.
5. Preconditions: provider has a window variant (else `not_measured`).
6. Action: for the runtime variant, hash the provider configuration, call the account-usage function, hash again; read each window's percent, length and reset time.
7. Autonomy: A3.
8. Timeout / retry: 15 s per provider / 1 retry.
9. Idempotency: read-only.
10. Postconditions: window snapshots, or a fetch error per provider.
11. Verification: config hash unchanged; values validated in S6.
12. Failure routes: fetch error → keep last good, set `last_fetch_error`; config hash changed → stop the variant for this cycle and later cycles until an operator re-enables it, `config_changed` critical alert.
13. Escalation: config change → operator.
14. Evidence: hashes compared (not the config), snapshot values.

**S4 — Snapshot credit balances**
1. Objective: balance and runway per pay-as-you-go provider.
2. Capability: `nova.quota.balance.fetch` (`provider-balance-api`, else `ledger-derived`).
3. Input mapping: pay-as-you-go providers; 7-day average daily spend from the ledger.
4. Dependencies: S2.
5. Preconditions: provider has a balance variant (else `not_measured`).
6. Action: read the balance (or compute purchases minus recorded spend, labelled `ledger_estimate`); compute runway.
7. Autonomy: A3.
8. Timeout / retry: 15 s per provider / 1 retry.
9. Idempotency: read-only.
10. Postconditions: balance snapshots.
11. Verification: basis recorded; runway null when average spend is 0.
12. Failure routes: fetch error → keep last good; ledger unavailable → `not_measured` for ledger-derived providers this cycle.
13. Escalation: none here.
14. Evidence: balance, basis, runway.

**S5 — Scan exhaustion signals**
1. Objective: catch a provider that already ran dry.
2. Capability: `nova.quota.signals.scan.default`.
3. Input mapping: recent call errors and job failures over the last 60 minutes.
4. Dependencies: S2.
5. Preconditions: error sources readable.
6. Action: count insufficient-credit, quota-exceeded and payment-required errors per provider.
7. Autonomy: A3.
8. Timeout / retry: 10 s / 1 retry.
9. Idempotency: read-only.
10. Postconditions: signals per provider.
11. Verification: each signal names its source.
12. Failure routes: source unreadable → no signals, disclosed in the cycle record.
13. Escalation: none here.
14. Evidence: signal counts.

**S6 — Validate snapshots**
1. Objective: reject impossible values before they are stored.
2. Capability: validation within `nova.quota.evaluate`.
3. Input mapping: S3–S5 outputs.
4. Dependencies: S3, S4, S5.
5. Preconditions: snapshots present.
6. Action: percent within 0–100; reset time in the future (or within the current window); window length positive; balance numeric; currency present when a balance is present.
7. Autonomy: A3.
8. Timeout / retry: 2 s / none.
9. Idempotency: pure.
10. Postconditions: valid snapshots; invalid ones turned into fetch errors.
11. Verification: every stored value passes its range check.
12. Failure routes: invalid → treated as a fetch error (last good kept).
13. Escalation: none.
14. Evidence: validation results.

**S7 — Upsert snapshots**
1. Objective: persist live values without touching policy.
2. Capability: `nova.quota.store.upsert_snapshot`.
3. Input mapping: valid snapshots; fetch errors.
4. Dependencies: S6.
5. Preconditions: not `dry_run`.
6. Action / side effects: write live columns only; on error update only `last_fetch_error`.
7. Autonomy: A3.
8. Timeout / retry: 10 s / 1 retry.
9. Idempotency: upsert keyed by provider and window.
10. Postconditions: store refreshed.
11. Verification: re-read equals written; policy text unchanged (hash).
12. Failure routes: write fails → `failed`; the view turns `stale` after 3 hours.
13. Escalation: 6 consecutive failures → operator.
14. Evidence: rows written, policy hash.

**S8 — Evaluate levels and fallback independence**
1. Objective: one level per backend and chain verdicts.
2. Capability: `nova.quota.evaluate.assess`.
3. Input mapping: snapshots (with age), signals, thresholds, registry chains.
4. Dependencies: S7.
5. Preconditions: snapshots stored.
6. Action: apply the levels table; worst window wins (Q4); a recent exhaustion signal → `exhausted`; snapshots older than 3 hours count as `stale` for chain checks; apply Q6.
7. Autonomy: A3.
8. Timeout / retry: 2 s / none.
9. Idempotency: pure.
10. Postconditions: `evaluation@0.1.0` per backend.
11. Verification: each level carries reasons with the numbers.
12. Failure routes: rule error → backend `not_measured` with reason.
13. Escalation: see S9.
14. Evidence: evaluations.

**S9 — Alert on crossings**
1. Objective: one clear alert per real crossing.
2. Capability: `nova.notify.owner.send`.
3. Input mapping: evaluations vs previous levels; dedupe keys.
4. Dependencies: S8.
5. Preconditions: not `dry_run`.
6. Action / side effects: alert on upward crossings (to `warn`, `critical`, `exhausted`), on "no independent fallback", "no working backend" and `config_changed`; send one recovery notice on return to `ok`; hold `warn` during quiet hours.
7. Autonomy: A3.
8. Timeout / retry: 15 s / 2 retries.
9. Idempotency: `dedupe_key`; send-then-mark.
10. Postconditions: alerts delivered or pending.
11. Verification: delivery receipts.
12. Failure routes: channel down → pending for the next cycle.
13. Escalation: `critical` and `exhausted` repeat once every 6 hours while unresolved (`TBD`).
14. Evidence: alert records.

**S10 — Close the cycle**
1. Objective: an auditable cycle record.
2. Capability: `nova.quota.store.record_cycle` (close).
3. Input mapping: per-provider fetch results, levels, alerts.
4. Dependencies: S9.
5. Preconditions: cycle open.
6. Action / side effects: write the result; release the lock.
7. Autonomy: A3.
8. Timeout / retry: 5 s / 1 retry.
9. Idempotency: by `cycle_id`.
10. Postconditions: cycle closed.
11. Verification: every configured provider has an entry.
12. Failure routes: close fails → lock expires after 30 minutes.
13. Escalation: a provider in `error` for 6 cycles → operator.
14. Evidence: `cycle_result`.

**S11 — Serve reads with the stale rule**
1. Objective: every reader sees honest headroom.
2. Capability: `nova.quota.store.list`.
3. Input mapping: optional provider filter.
4. Dependencies: stored snapshots (any cycle).
5. Preconditions: reader authorised.
6. Action: compute `display_state`: `stale` when the newest snapshot is older than 3 hours; otherwise the evaluated level; show the last known value only with its age.
7. Autonomy: A3.
8. Timeout / retry: 5 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: `quota-view@0.1.0` rows.
11. Verification: no row older than 3 hours displays anything but `stale`; no null shown as 0.
12. Failure routes: store unreadable → error, never an empty "all fine".
13. Escalation: none.
14. Evidence: read timestamp.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Observe only | runtime config hash unchanged after each call | S3 | equal | stop variant; critical alert |
| Read-only | no write or purchase call made to any provider | adapter log | none | stop; escalate |
| Value ranges | percent 0–100, reset in future, numeric balance | S6 | pass | treat as fetch error |
| Last good kept | failed fetch leaves previous values | S7 re-read | unchanged | fix write path |
| Policy intact | policy text hash unchanged by writes | S7 | equal | restore from configuration; escalate |
| Stale rule | rows older than 3 h display `stale` | S11 | always | treat as defect |
| Null not zero | unknown values are null | S7, S11 | always | treat as defect |
| Exhaustion override | recent signal → `exhausted` regardless of balance | S8 | always | re-evaluate |
| Alert discipline | one alert per crossing; recovery once | S9 | pass | suppress duplicates |
| Output schema | outputs valid | validator | valid | fix fields; never fake values |

A usage call returning data proves only that the source answered, not that the number is current or that the runtime is unchanged.

## J. Self-correction and recovery

- **Failure classes:** lock contention, provider fetch error, unsupported provider, invalid values, config hash change, ledger unavailable, store write failure, alert delivery failure.
- **Permitted corrections:** one retry per provider per cycle; keep last good values; fall back from the balance API to the ledger-derived estimate only where configured (labelled); queue alerts.
- **Forbidden corrections:** writing zeros; hiding `stale`; changing thresholds to silence alerts; switching or re-routing models; retrying a runtime call that changed configuration.
- **Maximum correction cycles:** 1 per provider per cycle.
- **Reverification required after correction:** yes — S6 validation on every snapshot.
- **Stop conditions:** config hash changed (for that variant); store unavailable.
- **Recovery or compensation:** the next clean cycle refreshes values; recovery notices close alerts.
- **What must be reported:** fetch errors per provider, stale providers, signals seen, alerts sent or pending.

## K. Independent challenge

- **Assumptions to challenge:** the reported percent reflects the window the assistant actually draws from; the balance includes pending charges; the fallback really works when the primary is out.
- **Alternative approach:** a tiny periodic health call through each fallback would prove it answers (it costs tokens and is outside this recipe).
- **Contradictory evidence:** positive balance with recent insufficient-credit errors; low percent with rate-limit errors.
- **Could mislead:** a stale "12 % used" read as current; a ledger estimate shown as a provider balance; a fallback chain that loops back to the same account.
- **Separate verifier:** exhaustion signals come from call outcomes, independent of the provider's usage figures; the read path enforces staleness independently of the collector.
- **Findings requiring disclosure:** estimates, `not_measured` backends, stale providers, missing independent fallback.

## L. Human escalation

- **Triggers:** `critical`; `exhausted`; "no working backend"; "no independent fallback"; config hash changed; a provider fetch failing for 3 hours.
- **Why the agent cannot decide:** buying credits, changing plans and routing are owner decisions.
- **Information presented:** backend, role, window or balance, percent or runway, reset time, signals, fallback chain status.
- **Decision requested:** top up, wait for the reset, add an independent fallback, or change routing (all done by a human).
- **Options and consequences:** for example, waiting for a reset leaves scheduled jobs failing until then.
- **Safe behaviour while waiting:** keep monitoring; no automatic changes.
- **If no response:** `critical` and `exhausted` repeat every 6 hours (`TBD`), `warn` does not repeat.
- **Resume:** automatic; levels recover on the next clean snapshot.

## M. Completion conditions

Mandatory: `cycle_result` valid; every configured provider has a fetch result; snapshots validated before storing; last good kept on failure; policy text untouched; stale rule enforced at read time; alerts once per crossing; limitations disclosed; evidence references present.

- `completed` — all measurable providers fetched, stored, evaluated and alerts handled.
- `partially_completed` — at least one provider fetch failed or a variant was stopped, or alerts are pending delivery.
- `blocked` — registry unreadable or no read access to any source.
- `failed` — the snapshot store write failed.
- `cancelled` — skipped because another cycle held the lock, or stopped by the operator.

## N. Final output contract

Return: run (cycle) ID; recipe ID and version; status; one-line outcome (for example "assistant-plan 78 % of weekly window, resets Thu 06:00; payg-gateway runway 1.6 days"); `cycle_result`; step statuses S1–S11; verification results and failed checks; evidence references (cycle record, config hash checks, alert IDs); approvals and scope (none at run time); side effects (snapshot writes, alerts) and reconciliation status; corrections; unresolved challenges; limitations (estimates, not-measured backends); escalation state; learning candidate. Never include credentials, account identifiers or configuration contents.

## O. Learning and revision

- **Outcome metrics:** lead time between `warn` and exhaustion, alerts per crossing (should be 1), share of `stale` and `not_measured`, false exhaustion signals.
- **Evidence retained:** cycle records, snapshot history, alert records.
- **Data minimisation:** percentages, balances and timestamps only.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}`.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review; a revision can never grant purchase or routing permissions or weaken Q1.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (fixture subscription and prepaid provider) | windows and balance stored; levels correct | UNVERIFIED on Skill Port; core flow observed in the reference implementation |
| Invalid input (critical below warn) | defaults with a warning | TBD |
| Missing input (no thresholds) | 75 / 90 / 3 h defaults | TBD |
| Missing capability (no window variant for a provider) | `not_measured`, never 0 % | TBD |
| Permission denied (usage scope missing) | fetch error; last good kept | TBD |
| Unauthorized scope (adapter attempts a billing write) | refused; escalation | TBD |
| Tenant isolation | tenant A cannot read tenant B's snapshots | TBD |
| Timeout (slow provider) | one retry, then fetch error; others unaffected | TBD |
| Duplicate / replay (two cycles at once; same crossing twice) | single-flight; one alert | TBD |
| Thresholds (74, 75, 89, 90, 100 %) | ok, warn, warn, critical, exhausted | TBD |
| Worst window (short window 92 %, weekly 40 %) | critical | TBD |
| Balance runway (10, 5, 1 day; balance 0) | ok, warn, critical, exhausted | TBD |
| Exhaustion signal with positive balance | exhausted, both facts shown | TBD |
| Fallback chain on the same billing account | warn "no independent fallback" | TBD |
| Primary critical and fallback exhausted | critical "no working backend" | TBD |
| False success (collector stopped 4 hours) | all rows `stale`, none "0 %" | TBD |
| Failed verification (percent 140 returned) | rejected as fetch error | TBD |
| Config hash changed by a runtime call | variant stopped; `config_changed` alert | TBD |
| Correction budget exhausted (fetch fails 6 cycles) | escalation | TBD |
| Escalation and recovery (critical then reset) | one alert, one recovery notice | TBD |
| Cancellation (lock held) | `cancelled`, nothing written | TBD |
| Rollback (snapshot write fails) | previous values intact | TBD |
| Output schema and status | valid outputs, correct status | TBD |
| Compatibility / regression (provider changes its usage format) | S6 rejects bad values; no zeros written | TBD |

All tests use fixture providers and a sandbox store; never real purchases or production routing. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, schema registry and how read-only account access is granted per provider — TBD (owner: publisher, with Skill Port).
- Balance floors, runway days, quiet hours and repeat interval — TBD (owner: operator).
- Many providers have no balance endpoint; the ledger-derived estimate depends on the operator recording purchases.
- Whether usage reads consume billable quota differs by provider — TBD.
- Snapshot retention — TBD (owner: operator).
