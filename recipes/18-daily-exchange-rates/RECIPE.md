# Daily exchange rates

## A. Identity and ownership

- **Recipe ID:** `nova.finance.daily-exchange-rates` (proposed until assigned by Skill Port)
- **Name:** Daily exchange rates
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** finance
- **Tags:** fx, exchange-rates, currency-conversion, cross-check, staleness, source-attribution, reference-data · Vedaxi task type: none (no supported task type covers this today)
- **Summary:** Once a day, the agent fetches mid-market exchange rates for the user's currencies from a primary source and cross-checks them against an independent second source. It stores an append-only, source-attributed snapshot that other recipes use for conversions. Every snapshot carries a status — fresh, stale, expired or unverified — so no consumer ever presents an old or unconfirmed rate as current.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** Two independent HTTPS rate sources covering the configured currencies (operator-chosen; one may need an API key); a snapshot store; the owner channel for alerts. Consumers: `nova.finance.subscription-audit`, `nova.finance.net-worth-overview`. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — a daily refresh from a free mid-market source, cached with its base currency, cross-checked against an independent source with a 0.5 % divergence alert, has run in Nova's reference implementation; values were spot-checked against a third public reference (catalogue status: Built). Not every control described here has evidence yet (Section P lists the tests still needed). Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.finance.fx` | Capability | Daily exchange-rate snapshots for conversions | low | UNVERIFIED |
| `nova.finance.fx.fetch_primary` | Operation (external read) | Fetch the latest rates for a base currency from the primary source | low | UNVERIFIED |
| `nova.finance.fx.fetch_secondary` | Operation (external read) | Fetch the same currencies from an independent source | low | UNVERIFIED |
| `nova.finance.fx.store_snapshot` | Operation (create, append-only) | Store a snapshot with per-currency cross-check results and attribution | low | UNVERIFIED |
| `nova.finance.fx.get_rates` | Operation (read) | Return the latest (or as-of) snapshot with a computed status | low | UNVERIFIED |
| `nova.notify.owner` | Capability | Owner channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Divergence and staleness alerts | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.finance.fx.fetch_primary.keyless` — a free daily mid-market endpoint that needs no key (as in the reference). Default.
- `nova.finance.fx.fetch_primary.keyed` — a provider that needs `secret_ref://fx-primary`. Selected by operator configuration only.
- `nova.finance.fx.fetch_secondary.default` — an independent source with different upstream data. It must not be a mirror of the primary.
- `nova.finance.fx.store_snapshot.secondary-only` — eligible only when the primary fetch or validation fails. It stores the secondary's rates with `status = unverified` and attribution to the secondary. Rates are read-only reference data, so this fallback adds no side effect, and the label is never hidden.
- `nova.finance.fx.get_rates.default` — computes status at read time from the snapshot's age and cross-check result.

**Bindings:** `binding://nova/finance/fx-primary`, `binding://nova/finance/fx-secondary`, `binding://nova/finance/fx-store`, `binding://nova/finance/fx-get`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: a daily maintenance job fetches rates, caches them as a JSON value with base currency and rates map behind a database function (`set` / `get`), cross-checks them against a second public source and sends a chat alert on divergence above 0.5 %. Provider names and URLs are operator configuration and are not part of this recipe.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| FX snapshot | `schema://nova/daily-exchange-rates/fx-snapshot@0.1.0` | installation (reference data, no personal data) | create (append-only), read |
| Provider response | transient | one run | read |

**Authorization requirements (declared, never a grant):** `finance:fx:write`, `finance:fx:read`, `net:egress:fx-sources` (outbound HTTPS to the two configured hosts only), `notify:owner`. Secrets: `secret_ref://fx-primary` (keyed variant only).

**Policy / trust:** risk class `low`. Provider responses are untrusted input: they are validated, bounds-checked and cross-checked. Rates are informational mid-market values, not bank or card rates, and are never used to execute trades or payments.

## C. Intent and applicability

- **User problem:** Accounts and subscriptions in several currencies cannot be added up without trustworthy, dated exchange rates.
- **Desired outcome:** One verified snapshot per day, with a clear source and age, that every finance recipe converts with the same way.
- **Trigger or entry conditions:** Daily schedule (default 06:30 in the installation time zone), or "refresh exchange rates now".
- **Applicable situations:** Net-worth totals, subscription totals, any display conversion.
- **Out-of-scope situations:** Trading or exchanging money; quoting a bank's buy/sell rate; historical backfill beyond what the source provides; crypto prices.
- **Required inputs:** `base_currency`, `currencies` (see Section D).
- **Prerequisites:** Two independent sources configured; outbound HTTPS allowed to their hosts.
- **Required capabilities:** `nova.finance.fx`, `nova.notify.owner` (proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port; every binding is a dependency to resolve before activation.
- **Assumptions:** Verified in the reference implementation: a daily keyless mid-market source with the needed currencies, an independent cross-check with a 0.5 % alert threshold, and a cached base-currency snapshot. Not every control described here has evidence yet (Section P lists the tests still needed).
- **Ambiguity policy:** Use a documented safe default. A rate that cannot be confirmed is stored as `unverified`, and consumers must not show it as a plain converted total.
- **Vedaxi task type:** none.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `base_currency` | ISO 4217 | yes | operator/user | — | `blocked`. |
| `currencies` | array of ISO 4217, 1–50 | yes | derived from the user's accounts + user | — | Empty → `blocked`. |
| `divergence_tolerance_pct` | number 0.1–2.0 | no | operator | 0.5 | Clamp. |
| `max_daily_change_pct` | number 2–50 | no | operator | 10 | Clamp. |
| `fresh_hours` | integer 24–72 | no | operator | 36 | Clamp. |
| `expire_days` | integer 2–30 | no | operator | 7 | Clamp. |
| `sources` | `{primary:{binding, display_name}, secondary:{binding, display_name}}` | yes | operator | — | Same upstream for both → `blocked` ("not independent"). |

```json
{"$id":"schema://nova/daily-exchange-rates/config@0.1.0","type":"object","required":["base_currency","currencies","sources"],
 "properties":{"base_currency":{"type":"string","pattern":"^[A-Z]{3}$"},"currencies":{"type":"array","minItems":1,"maxItems":50,"items":{"type":"string","pattern":"^[A-Z]{3}$"}},
 "divergence_tolerance_pct":{"type":"number","minimum":0.1,"maximum":2},"max_daily_change_pct":{"type":"number","minimum":2,"maximum":50},
 "fresh_hours":{"type":"integer","minimum":24,"maximum":72},"expire_days":{"type":"integer","minimum":2,"maximum":30},
 "sources":{"type":"object","required":["primary","secondary"],"properties":{"primary":{"type":"object","required":["binding","display_name"]},"secondary":{"type":"object","required":["binding","display_name"]}}}}}
```

```json
{"$id":"schema://nova/daily-exchange-rates/fx-query@0.1.0","type":"object","properties":{"as_of":{"type":["string","null"],"format":"date-time"},"currencies":{"type":"array","items":{"type":"string","pattern":"^[A-Z]{3}$"}}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `fx_snapshot` | `schema://nova/daily-exchange-rates/fx-snapshot@0.1.0` | see schema | snapshot store; consumers via `get_rates` | public reference data | validation + cross-check + read-back |
| `fx_run` | `schema://nova/daily-exchange-rates/fx-run@0.1.0` | `run_id`, `status`, `snapshot_id` | run record | none | read-back |

```json
{"$id":"schema://nova/daily-exchange-rates/fx-snapshot@0.1.0","type":"object",
 "required":["snapshot_id","base","rates","source","provider_as_of","fetched_at","cross_check","status"],
 "properties":{"snapshot_id":{"type":"string"},"base":{"type":"string","pattern":"^[A-Z]{3}$"},
 "rates":{"type":"object","description":"units of currency X per 1 unit of base; base itself = \"1\"","additionalProperties":{"type":"string","pattern":"^[0-9]{1,9}(\\.[0-9]{1,10})?$"}},
 "source":{"type":"object","required":["display_name","role"],"properties":{"display_name":{"type":"string"},"role":{"enum":["primary","secondary_only"]},"terms_note":{"type":"string"}}},
 "provider_as_of":{"type":"string","format":"date-time"},"fetched_at":{"type":"string","format":"date-time"},
 "cross_check":{"type":"object","required":["performed"],"properties":{"performed":{"type":"boolean"},"secondary_display_name":{"type":["string","null"]},
  "max_divergence_pct":{"type":["string","null"]},"per_currency":{"type":"object","additionalProperties":{"enum":["agree","diverge","missing"]}}}},
 "status":{"enum":["fresh","stale","expired","unverified"],"description":"computed at read time from age and cross_check"}}}
```

Supporting schemas (one per line):

```json
{"$id":"schema://nova/daily-exchange-rates/provider-rates@0.1.0","type":"object","required":["source_role","base","rates","provider_as_of"],"description":"transient parsed provider response","properties":{"source_role":{"enum":["primary","secondary"]},"base":{"type":"string","pattern":"^[A-Z]{3}$"},"rates":{"type":"object","additionalProperties":{"type":"string","pattern":"^[0-9]{1,9}(\\.[0-9]{1,10})?$"}},"provider_as_of":{"type":"string","format":"date-time"},"http_status":{"type":"integer"}}}
{"$id":"schema://nova/daily-exchange-rates/fx-run@0.1.0","type":"object","required":["run_id","status"],"properties":{"run_id":{"type":"string"},"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},"snapshot_id":{"type":["string","null"]},"snapshot_status":{"enum":["fresh","stale","expired","unverified",null]},"fallback_used":{"type":"boolean"},"alerts_sent":{"type":"integer","minimum":0},"limitations":{"type":"array","items":{"type":"string"}}}}
{"$id":"schema://nova/daily-exchange-rates/delivery@0.1.0","type":"object","required":["delivered"],"properties":{"delivered":{"type":"boolean"},"message_id":{"type":["string","null"]},"transition":{"enum":["fresh_to_stale","stale_to_expired","divergence","source_failure"]},"error":{"type":["string","null"]}}}
```

**Conversion contract (for consumers):** `amount_in_base = amount_X ÷ rates[X]`; `amount_in_Y = amount_X × rates[Y] ÷ rates[X]`. Decimal arithmetic only, rounded half-even at display. A consumer must show the snapshot's `source.display_name`, `provider_as_of` and `status`, and must not produce a plain converted total from an `expired` or `unverified` snapshot.

**Example (fictional)**

Config: `{"base_currency":"USD","currencies":["EUR","GBP","JPY"],"sources":{"primary":{"binding":"binding://nova/finance/fx-primary","display_name":"Primary rates provider"},"secondary":{"binding":"binding://nova/finance/fx-secondary","display_name":"Secondary rates provider"}}}`

Output: `{"snapshot_id":"fx_2026_11_03","base":"USD","rates":{"USD":"1","EUR":"0.921350","GBP":"0.789120","JPY":"151.2300"},"source":{"display_name":"Primary rates provider","role":"primary","terms_note":"mid-market, informational"},"provider_as_of":"2026-11-03T00:00:00Z","fetched_at":"2026-11-03T02:30:05Z","cross_check":{"performed":true,"secondary_display_name":"Secondary rates provider","max_divergence_pct":"0.086","per_currency":{"EUR":"agree","GBP":"agree","JPY":"agree"}},"status":"fresh"}`

## E. Constraints and budgets

- **Hard constraints:** rates are stored as decimal strings with explicit base semantics; snapshots are append-only (never overwritten or deleted); every snapshot carries source attribution, the provider's as-of time and the cross-check result; status is computed honestly (`fresh` only when cross-checked and ≤ `fresh_hours` old; `expired` beyond `expire_days`; `unverified` when single-source, divergent or beyond the daily-change bound); no personal data leaves the runtime (requests contain only currency codes); never execute trades or payments.
- **Soft constraints:** fetch shortly after the sources' daily update; prefer the primary's values when both agree.
- **Resource and tenant scope:** the installation's snapshot store.
- **Data sources allowed:** the two configured rate sources.
- **External transfers permitted:** HTTPS GET requests to the two configured hosts.
- **Time limit:** 2 minutes per run.
- **Cost or token budget:** two HTTP calls per run (plus retries); no LLM.
- **Maximum steps:** 9 (S1–S9).
- **Maximum retries and correction cycles:** each fetch 3 attempts with backoff (5 s, 30 s); correction cycles 1.
- **Rate limits and concurrency:** one run at a time; at most 4 manual refreshes per day (provider terms).
- **Retention and logging constraints:** snapshots kept for at least 400 days (enough for a 13-month audit; final value `TBD`); raw provider responses are not stored.
- **Prohibited actions:** scraping pages that forbid automated access; mixing values from two sources into one snapshot without labelling; editing a past snapshot.
- **Stop conditions:** both sources fail (keep the previous snapshot; status ages naturally); kill switch.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | FX snapshot (append-only) | primary or secondary-only validated | Allowed | read-back via `get_rates` |
| Read | snapshots | installation | Allowed | — |
| Update | snapshots | — | Not Allowed (append-only) | — |
| Delete | snapshots | — | Not Allowed (retention policy only, outside this recipe) | — |
| External side effect | HTTPS GET to rate sources | configured hosts only | Allowed | response validation |
| External side effect | alert to the owner | divergence, staleness, expiry, source failure | Allowed | receipt |

- **Idempotency:** snapshot key = `(base, provider_as_of, source role)`; a second run with the same provider timestamp returns the existing snapshot.
- **Expected side effects:** one snapshot per day; occasional alerts.
- **Compensation:** not applicable to stored values (append-only); a bad snapshot is superseded by a newer one and can be marked `unverified` by an operator action recorded in the audit log (outside this recipe).
- **Irreversible:** none of consequence.
- **Ambiguous outcomes:** if storing times out, read back by key before retrying.

## G. Autonomy and approval policy

- **Default autonomy level:** A3 — conditional autonomy (reference data, no personal data, no money, bounded recovery).
- **Planning level:** A3.
- **Execution level:** A3.
- **Verification level:** A3.
- **Recovery level:** A2 — bounded retries; secondary-only fallback labelled `unverified`.
- **Approval required for:** changing sources or tolerance (operator); nothing at run time.
- **Actions never permitted:** trading, payments, editing past snapshots, unlabelled source mixing.
- **Conditions requiring clarification:** none at run time.
- **Conditions requiring escalation:** divergence above tolerance on 2 consecutive days; snapshot older than `fresh_hours` (once) and older than `expire_days` (once); both sources failing 3 days running.
- **Safe state after stop, cancellation or escalation:** previous snapshots untouched; consumers see the honest status.

## H. Execution plan

**S1 — Validate configuration and take the lock**
1. Objective: a sane, independent configuration.
2. Capability: none (local validation).
3. Input mapping: config → `config@0.1.0`.
4. Dependencies: none.
5. Preconditions: kill switch off.
6. Action / side effects: validate; check that the two sources are configured as different upstreams; take lock `fx:<installation>`.
7. Autonomy: A3.
8. Timeout / retry: 5 s / none.
9. Idempotency: lock.
10. Postconditions: valid config.
11. Verification: schema valid; base not listed twice; currency codes unique.
12. Failure routes: invalid → `blocked`.
13. Escalation triggers: none.
14. Evidence: config hash.

**S2 — Fetch the primary source**
1. Objective: today's rates.
2. Capability: `nova.finance.fx.fetch_primary` (`keyless` or `keyed`).
3. Input mapping: `base_currency`.
4. Dependencies: S1.
5. Preconditions: egress allowed.
6. Action / side effects: one HTTPS GET; response held in memory.
7. Autonomy: A3.
8. Timeout / retry: 15 s / 3 attempts with backoff.
9. Idempotency: read-only.
10. Postconditions: a parsed response or a failure.
11. Verification: HTTP 200, JSON parse, TLS valid.
12. Failure routes: failure after retries → go to S4 and plan the secondary-only fallback.
13. Escalation: counted toward the 3-day source-failure rule.
14. Evidence: status code, provider timestamp.

**S3 — Validate the primary response**
1. Objective: reject broken or implausible data.
2. Capability: deterministic checks.
3. Input mapping: response, `currencies`, previous snapshot.
4. Dependencies: S2.
5. Preconditions: a response exists.
6. Action: base equals the configured base; every configured currency present; values positive, finite, ≤ 10 significant digits; provider timestamp ≤ now + 10 minutes and not older than the previous stored snapshot; per-currency change versus the previous snapshot ≤ `max_daily_change_pct` (larger → that currency flagged).
7. Autonomy: A3.
8. Timeout / retry: 1 s / none.
9. Idempotency: pure.
10. Postconditions: validated rates, with flags.
11. Verification: all checks pass, or flags recorded.
12. Failure routes: missing currency or bad base → treat the primary as failed; same timestamp as the stored snapshot → `completed` with no new snapshot ("no update yet").
13. Escalation: none.
14. Evidence: validation report.

**S4 — Fetch the secondary source**
1. Objective: an independent comparison.
2. Capability: `nova.finance.fx.fetch_secondary.default`.
3. Input mapping: `base_currency` (converted if the source uses a different base: `rate_X = src_X ÷ src_base`).
4. Dependencies: S2 (runs even if S2 failed).
5. Preconditions: egress allowed.
6. Action / side effects: one HTTPS GET.
7. Autonomy: A3.
8. Timeout / retry: 15 s / 3 attempts.
9. Idempotency: read-only.
10. Postconditions: secondary rates or a failure.
11. Verification: same structural checks as S3.
12. Failure routes: failure → the cross-check is `performed: false`, so the snapshot will be `unverified`.
13. Escalation: counted toward the source-failure rule.
14. Evidence: status code, provider timestamp.

**S5 — Cross-check**
1. Objective: confirm the primary's numbers.
2. Capability: deterministic comparison.
3. Input mapping: primary and secondary rates, tolerance.
4. Dependencies: S3, S4.
5. Preconditions: both present (otherwise mark `performed: false`).
6. Action: per currency, divergence = |p − s| ÷ p × 100; `agree` if ≤ tolerance, otherwise `diverge`; a currency absent from the secondary → `missing`.
7. Autonomy: A3.
8. Timeout / retry: 1 s / none.
9. Idempotency: pure.
10. Postconditions: cross-check object with `max_divergence_pct`.
11. Verification: recomputed in S8.
12. Failure routes: any `diverge`, any daily-change flag or any `missing` for a configured currency → the snapshot is `unverified`.
13. Escalation: divergence 2 days running → alert.
14. Evidence: per-currency results.

**S6 — Store the snapshot**
1. Objective: an append-only, attributed record.
2. Capability: `nova.finance.fx.store_snapshot` (default, or `secondary-only` when the primary failed).
3. Input mapping: chosen rates, attribution, `provider_as_of`, `fetched_at`, cross-check.
4. Dependencies: S5.
5. Preconditions: at least one validated source.
6. Action / side effects: insert one snapshot.
7. Autonomy: A3.
8. Timeout / retry: 10 s / 2 after read-back by key.
9. Idempotency: key `(base, provider_as_of, role)`.
10. Postconditions: snapshot stored.
11. Verification: insert returns the snapshot ID.
12. Failure routes: no validated source → no snapshot; `failed`; the previous snapshot stays in force and ages.
13. Escalation: none.
14. Evidence: snapshot ID.

**S7 — Evaluate staleness and alert**
1. Objective: tell the user once when rates become unreliable.
2. Capability: `nova.finance.fx.get_rates.default`; `nova.notify.owner.send`.
3. Input mapping: latest snapshot, thresholds, alert history.
4. Dependencies: S6 (or S1 when nothing new was stored).
5. Preconditions: none.
6. Action / side effects: compute the status; send one alert per transition (fresh → stale, stale → expired, persistent divergence, 3-day source failure), for example: "Exchange rates are 2 days old; converted totals are marked stale."
7. Autonomy: A3.
8. Timeout / retry: 15 s / 2.
9. Idempotency: alert key = `(transition, snapshot_id)`.
10. Postconditions: status known; alerts sent once.
11. Verification: receipt for each alert.
12. Failure routes: alert fails → retry next run (key unchanged).
13. Escalation: as listed in Section G.
14. Evidence: status, alert IDs.

**S8 — Verify read-back**
1. Objective: consumers will see what was stored.
2. Capability: `nova.finance.fx.get_rates.default`.
3. Input mapping: `snapshot_id`.
4. Dependencies: S6.
5. Preconditions: a new snapshot exists.
6. Action: read; recompute the cross-check and status independently.
7. Autonomy: A3.
8. Timeout / retry: 5 s / 1.
9. Idempotency: read-only.
10. Postconditions: verified.
11. Verification: rates equal the validated values; status equals the recomputed status; a conversion round trip (base → X → base) is exact to 6 decimal places.
12. Failure routes: mismatch → `partially_completed`; the snapshot is reported as suspect (consumers keep using the previous `fresh` one until it ages).
13. Escalation: none.
14. Evidence: verification result.

**S9 — Report**
1. Objective: an honest run record.
2. Capability: run record store.
3. Input mapping: results → `fx-run@0.1.0`.
4. Dependencies: S7, S8.
5. Preconditions: none.
6. Action / side effects: store the run; release the lock.
7. Autonomy: A3.
8. Timeout / retry: 5 s / 2.
9. Idempotency: `run_id`.
10. Postconditions: stored.
11. Verification: schema valid.
12. Failure routes: none.
13. Escalation: none.
14. Evidence: run ID.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Input validity | config valid; sources independent | validator | valid | `blocked` |
| Response sanity | base, completeness, positivity, timestamp monotonic | S3 | pass | fallback / unverified |
| Cross-check | divergence ≤ tolerance for every configured currency | S5, recomputed in S8 | all agree | `unverified` |
| Attribution | snapshot names its source, role and provider time | read-back | present | fix before store |
| Append-only | no snapshot updated or deleted | store audit | none | report defect |
| Status honesty | status matches age and cross-check | S8 recomputation | equal | fix status logic |
| No personal data egress | outbound requests contain only currency codes | request log | holds | stop, escalate |
| Output schema | snapshot and run valid | validator | valid | fix |

## J. Self-correction and recovery

- **Failure classes:** configuration, network or timeout, provider error, invalid response, divergence, implausible jump, store failure.
- **Permitted corrections:** bounded retries with backoff; secondary-only snapshot labelled `unverified`; skip storing when the provider has not updated.
- **Forbidden corrections:** averaging the two sources into an unlabelled value; widening the tolerance to pass; editing old snapshots; marking a single-source snapshot `fresh`.
- **Maximum correction cycles:** 1 per run.
- **Reverification required after correction:** yes — S8.
- **Stop conditions:** both sources failed; kill switch.
- **Recovery or compensation:** the next successful run supersedes; consumers rely on status.
- **What must be reported:** source failures, divergence, fallback use, staleness transitions.

## K. Independent challenge

- **Assumptions to challenge:** the two sources are truly independent (same upstream feed = no real check); the provider's timestamp reflects the data time, not the response time.
- **Alternative explanation:** a large divergence may be a real market move between the two sources' update times; the status stays `unverified` until they agree.
- **Contradictory evidence:** a third public reference, spot-checked during operator reviews.
- **Could mislead:** a mid-market rate shown as what the user would get at their bank. Consumers must label rates "mid-market, informational".
- **Separate verifier:** S8 recomputes divergence and status independently of S5.
- **Findings requiring disclosure:** unverified or stale status, fallback source, flagged currencies.

## L. Human escalation

- **Triggers:** divergence above tolerance 2 days running; staleness and expiry transitions; both sources failing 3 days running.
- **Why the agent cannot decide:** choosing a new source or accepting a divergent value is an operator or user judgement.
- **Information presented:** affected currencies, values from both sources, divergence, ages.
- **Decision requested:** keep waiting, switch sources (operator), or accept per-currency display only.
- **Options and consequences:** switching sources needs an operator configuration change and a fixture test.
- **Safe behaviour while waiting:** consumers show per-currency totals and status labels.
- **If no response:** status keeps ageing; converted totals stop once rates expire.
- **Resume:** the next run uses the current configuration.

## M. Completion conditions

Mandatory: the snapshot (if any) is valid, attributed and append-only; status computed honestly; cross-check recorded; alerts sent once per transition; evidence references present.

- `completed` — a new snapshot is stored and verified (any status), or the provider had no update yet.
- `partially_completed` — stored but verification found a mismatch, or an alert failed.
- `blocked` — invalid or non-independent configuration.
- `failed` — no validated source; nothing stored.
- `cancelled` — kill switch or lock held.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome ("EUR, GBP, JPY updated; cross-checked; fresh"); `fx_snapshot` reference and status; step statuses S1–S9; verification results; evidence references (snapshot ID, provider timestamps, cross-check); approvals and scope (configuration version); side effects (snapshot stored, alerts); corrections (fallback used); unresolved challenges; limitations; escalation state; learning candidate. No secrets or API keys.

## O. Learning and revision

- **Outcome metrics:** daily success rate per source, divergence distribution, fallback frequency, time in each status, alert count.
- **Evidence retained:** snapshots (retention ≥ 400 days) and run records.
- **Data minimisation:** no raw provider responses; no personal data involved.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}` — for example "secondary misses currency X on weekends → configure a third source".
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review; source changes need operator sign-off.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (mocked sources that agree) | `fresh` snapshot stored | UNVERIFIED on Skill Port; daily refresh + cross-check observed in the reference implementation |
| Invalid input (bad currency code) | `blocked` | TBD |
| Missing input (no currencies) | `blocked` | TBD |
| Same upstream configured twice | `blocked` (not independent) | TBD |
| Missing capability (secondary binding absent) | snapshot `unverified` | TBD |
| Permission denied (egress to source host blocked) | fetch fails; fallback or `failed` | TBD |
| Unauthorized scope (request to a non-configured host) | refused by egress policy | TBD |
| Tenant isolation | not applicable — reference data with no personal content; installation-scoped store tested instead | TBD |
| Divergence 1 % | `unverified`; alert after the second day | TBD |
| Daily jump 20 % | currency flagged; `unverified` | TBD |
| Primary down | secondary-only snapshot, `unverified` | TBD |
| Both down | no snapshot; previous one ages; alerts at transitions | TBD |
| Timeout (store) | read-back by key; no duplicate | TBD |
| Duplicate / replay (same provider timestamp) | existing snapshot returned | TBD |
| False success (200 with an empty rates map) | treated as a failed source | TBD |
| Failed verification (status mismatch) | `partially_completed` | TBD |
| Correction budget exhausted | safe stop | TBD |
| Escalation (expiry transition) | one alert | TBD |
| Cancellation (kill switch) | `cancelled` | TBD |
| Rollback / compensation | not applicable — append-only reference data; supersession tested instead | TBD |
| Output schema and status | valid snapshot and run | TBD |
| Compatibility / regression (provider changes format) | validation fails safely; alert | TBD |

All tests use mocked or recorded provider responses with fictional values; no real accounts are involved. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Which two providers to use, and their terms of use (attribution, rate limits, commercial use) — owner: operator.
- Some currencies are covered by only one free provider; for those the snapshot can never be cross-checked, so it stays `unverified`. A third source is needed (owner: operator).
- Mid-market rates differ from bank and card rates; totals are indicative only.
- Weekend and holiday data is usually unchanged; the "no update yet" path handles this.
- The retention period above 400 days — TBD (owner: operator).
