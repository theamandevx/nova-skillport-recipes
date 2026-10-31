# Net worth overview

## A. Identity and ownership

- **Recipe ID:** `nova.finance.net-worth-overview` (proposed until assigned by Skill Port)
- **Name:** Net worth overview
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** finance
- **Tags:** net-worth, balances, multi-currency, fx-conversion, read-only, stale-data · Vedaxi task type: none (no supported task type covers this today)
- **Summary:** On request, the agent reads every account balance from the finance hub. The hub mirrors the user's external personal-finance app over REST, and that app is the source of truth. The agent converts each native balance to one reporting currency with the latest verified exchange rates and returns assets, amounts owed and net worth, with per-currency subtotals. It is strictly read-only, and it flags anything stale: cached balances, a provider sync in progress, old balances, or stale or unverified rates.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** The external personal-finance app's REST API (read accounts); the finance hub's account list (kinds, credit limits) and cached balances; exchange-rate snapshots from `nova.finance.daily-exchange-rates`; a conversation or dashboard surface. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — live balances from the external app, kind grouping, available credit for cards, and a consolidated total converted with daily rates have run in Nova's reference implementation, both on a dashboard page and through an assistant query (catalogue status: Built). Not every control described here has evidence yet (Section P lists the tests still needed). Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.external_finance_app.accounts` | Capability | Live account balances from the external personal-finance app | medium | UNVERIFIED |
| `nova.external_finance_app.accounts.list` | Operation (read) | Return accounts with native currency, balance and sync signal | medium | UNVERIFIED |
| `nova.finance.hub` | Capability | Finance-hub account metadata and cached balances | medium | UNVERIFIED |
| `nova.finance.hub.accounts_list` | Operation (read) | Kinds, credit limits, active flag and mapping to external accounts | low | UNVERIFIED |
| `nova.finance.hub.cached_balances` | Operation (read) | Last recorded balance per account (seed + synced records) with as-of | medium | UNVERIFIED |
| `nova.finance.fx` | Capability | Exchange-rate snapshots | low | UNVERIFIED |
| `nova.finance.fx.get_rates` | Operation (read) | Latest snapshot with status | low | UNVERIFIED |
| `nova.finance.networth` | Capability | Conversion and aggregation | low | UNVERIFIED |
| `nova.finance.networth.compute` | Operation (read, compute) | Convert, aggregate and flag; no writes | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.external_finance_app.accounts.list.live-rest` — authenticated GET of the accounts list. It honours the provider's sync-in-progress signal and rate-limit headers. Default.
- `nova.finance.hub.cached_balances.default` — eligible only when the live call fails, is rate-limited or reports a sync in progress. It returns the hub's last recorded balance per account with its as-of time. Both sources are read-only, so the fallback adds no side effect. Every number from it is labelled **cached**, never live.
- `nova.finance.networth.compute.default` — decimal arithmetic; only the default variant is supported.
- `nova.finance.fx.get_rates.default` — status `fresh | stale | expired | unverified`.

**Bindings:** `binding://nova/external-finance-app/accounts`, `binding://nova/finance/hub-accounts`, `binding://nova/finance/hub-cached-balances`, `binding://nova/finance/fx-get`, `binding://nova/finance/networth-compute` — all `status: UNVERIFIED`. Reference implementation: a server-side dashboard component calls the external app's REST API with a server-held token (never exposed to the browser), joins account kinds and credit limits from the hub, and converts with the cached daily rates. A database summary function serves cached balances (seed balance plus synced records) to the assistant.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| External account | `schema://nova/net-worth-overview/external-account@0.1.0` | the user's external app account | read |
| Hub account metadata | `schema://nova/net-worth-overview/hub-account@0.1.0` | the requesting user | read |
| Cached balance | `schema://nova/net-worth-overview/cached-balance@0.1.0` | the requesting user | read |
| FX snapshot | `schema://nova/daily-exchange-rates/fx-snapshot@0.1.0` | reference data | read |
| Overview | `schema://nova/net-worth-overview/overview@0.1.0` | the requesting user | returned (not stored by default) |

**Authorization requirements (declared, never a grant):** `external_finance_app:accounts:read`, `finance:hub:accounts:read`, `finance:hub:balances:read`, `finance:fx:read`. External app token: `secret_ref://finance-app-api` (server-side only). Database role: `secret_ref://finance-db-role`.

**Policy / trust:** risk class `medium` (a full view of the user's finances; read-only). Output goes only to the requesting user. Shared screens should support a privacy blur, which is a UI responsibility.

## C. Intent and applicability

- **User problem:** Money is spread across banks, cash, cards, loans and currencies, so the user has no single honest number for what they are worth.
- **Desired outcome:** One net-worth figure in the reporting currency, with assets, debts, per-currency subtotals and clear flags whenever any input is not current.
- **Trigger or entry conditions:** "What's my net worth?", "show my balances", or opening the overview page.
- **Applicable situations:** Personal overview; checking available credit; before a big purchase.
- **Out-of-scope situations:** Moving money; changing balances or accounts; investment valuation beyond the balance the external app reports; tax advice; forecasting.
- **Required inputs:** `reporting_currency` (see Section D).
- **Prerequisites:** The external app connection authorized; hub account metadata configured (kinds, limits); FX snapshots available for multi-currency totals.
- **Required capabilities:** see Section B (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port; every binding is a dependency to resolve before activation.
- **Assumptions:** Verified in the reference implementation: live balances as the source of truth, cached balances (seed + synced records) for the assistant, kind grouping, available credit = limit + balance, and conversion with daily rates. Not every control described here has evidence yet (Section P lists the tests still needed).
- **Ambiguity policy:** Use a documented safe default. Accounts without a known kind are shown as `unknown` and flagged; balances without a rate are listed in native currency and excluded from the converted total.
- **Vedaxi task type:** none.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `reporting_currency` | ISO 4217 | yes | user profile | — | Ask once; else per-currency subtotals only. |
| `timezone` | IANA zone | no | profile | UTC | Used for "as of" display. |
| `include_kinds` | array of kinds | no | user | all active kinds | Unknown kinds ignored, reported. |
| `stale_after_days` | integer 1–30 | no | user | 7 | Clamp. |
| `drift_check` | boolean | no | operator | `true` | Compare live with cached balances when both are available. |

```json
{"$id":"schema://nova/net-worth-overview/request@0.1.0","type":"object","required":["reporting_currency"],
 "properties":{"reporting_currency":{"type":"string","pattern":"^[A-Z]{3}$"},"timezone":{"type":"string"},
 "include_kinds":{"type":"array","items":{"enum":["bank","cash","ewallet","card","bnpl","loan","receivable","investment","other"]}},
 "stale_after_days":{"type":"integer","minimum":1,"maximum":30},"drift_check":{"type":"boolean"}}}
{"$id":"schema://nova/net-worth-overview/external-account@0.1.0","type":"object","required":["external_id","name","currency","balance"],
 "properties":{"external_id":{"type":"string"},"name":{"type":"string","maxLength":80},"currency":{"type":"string","pattern":"^[A-Z]{3}$"},
 "balance":{"type":"string","pattern":"^-?[0-9]{1,13}(\\.[0-9]{1,4})?$"},"last_modified":{"type":["string","null"],"format":"date-time"},"archived":{"type":"boolean"}}}
{"$id":"schema://nova/net-worth-overview/external-accounts-result@0.1.0","type":"object","required":["accounts","sync_in_progress","fetched_at"],
 "properties":{"accounts":{"type":"array","items":{"$ref":"schema://nova/net-worth-overview/external-account@0.1.0"}},"sync_in_progress":{"type":"boolean"},"fetched_at":{"type":"string","format":"date-time"},"rate_limited":{"type":"boolean"}}}
{"$id":"schema://nova/net-worth-overview/hub-account@0.1.0","type":"object","required":["account_ref","kind","currency","active"],
 "properties":{"account_ref":{"type":"string"},"external_id":{"type":["string","null"]},"display_name":{"type":"string","maxLength":80},
 "kind":{"enum":["bank","cash","ewallet","card","bnpl","loan","receivable","investment","other"]},"currency":{"type":"string","pattern":"^[A-Z]{3}$"},
 "credit_limit":{"type":["string","null"],"pattern":"^[0-9]{1,13}(\\.[0-9]{1,4})?$"},"last4":{"type":["string","null"],"pattern":"^[0-9]{4}$"},"active":{"type":"boolean"}}}
{"$id":"schema://nova/net-worth-overview/hub-account-query@0.1.0","type":"object","properties":{"include_kinds":{"type":"array","items":{"type":"string"}},"active_only":{"type":"boolean"}}}
{"$id":"schema://nova/net-worth-overview/cached-balance@0.1.0","type":"object","required":["account_ref","currency","balance","as_of"],
 "properties":{"account_ref":{"type":"string"},"currency":{"type":"string","pattern":"^[A-Z]{3}$"},"balance":{"type":"string","pattern":"^-?[0-9]{1,13}(\\.[0-9]{1,4})?$"},
 "as_of":{"type":"string","format":"date-time"},"basis":{"const":"seed_plus_synced_records"},"unsynced_records":{"type":"integer","minimum":0}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `overview` | `schema://nova/net-worth-overview/overview@0.1.0` | see schema | reply in the conversation or dashboard view | highly sensitive financial data; owner only | independent recompute (Section I) |

```json
{"$id":"schema://nova/net-worth-overview/overview@0.1.0","type":"object","required":["as_of","reporting_currency","source_mode","by_currency","accounts","fx","flags","status"],
 "properties":{"as_of":{"type":"string","format":"date-time"},"reporting_currency":{"type":"string","pattern":"^[A-Z]{3}$"},
 "status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},"source_mode":{"enum":["live","cached","mixed"]},
 "totals":{"type":["object","null"],"properties":{"assets":{"type":"string","pattern":"^[0-9]{1,13}(\\.[0-9]{1,4})?$"},"owed":{"type":"string","pattern":"^-?[0-9]{1,13}(\\.[0-9]{1,4})?$"},"net_worth":{"type":"string","pattern":"^-?[0-9]{1,13}(\\.[0-9]{1,4})?$"},"excluded_accounts":{"type":"integer","minimum":0}}},
 "by_currency":{"type":"object","additionalProperties":{"type":"string","pattern":"^-?[0-9]{1,13}(\\.[0-9]{1,4})?$"}},
 "accounts":{"type":"array","items":{"type":"object","required":["account_ref","name","kind","currency","balance","balance_source","flags"],"properties":{
  "account_ref":{"type":"string"},"name":{"type":"string"},"kind":{"type":"string"},"currency":{"type":"string","pattern":"^[A-Z]{3}$"},
  "balance":{"type":"string","pattern":"^-?[0-9]{1,13}(\\.[0-9]{1,4})?$"},"converted":{"type":["string","null"]},"available_credit":{"type":["string","null"]},
  "balance_source":{"enum":["live","cached"]},"balance_as_of":{"type":["string","null"],"format":"date-time"},
  "flags":{"type":"array","items":{"enum":["cached","provider_syncing","stale_balance","no_rate","unknown_kind","unmapped","drift"]}}}}},
 "fx":{"type":"object","properties":{"snapshot_id":{"type":["string","null"]},"as_of":{"type":["string","null"],"format":"date-time"},"source":{"type":["string","null"]},"status":{"enum":["fresh","stale","expired","unverified","none"]}}},
 "flags":{"type":"array","items":{"type":"string"}},"limitations":{"type":"array","items":{"type":"string"}}}}
```

Reused schemas (acknowledged): `fx-snapshot@0.1.0` and `fx-query@0.1.0` are defined in Section D of `nova.finance.daily-exchange-rates` (folder `18-daily-exchange-rates`) and used unchanged, including the conversion contract.

**Rules:** converted = `balance × rates[reporting] ÷ rates[currency]`, with decimal arithmetic; totals come from unrounded values, rounded half-even to 2 places only for display. Assets = sum of positive converted balances; owed = sum of negative converted balances; net worth = assets + owed. Available credit (card, bnpl) = `credit_limit + balance`. A converted total is produced only when the FX status is `fresh` or `stale` (stale is labelled); `expired`, `unverified` or `none` → `totals = null`, and the overview shows per-currency subtotals only.

**Example (fictional; reporting currency USD; FX fresh, EUR 0.921350 per USD)**

Accounts: Example Bank checking 2,450.00 USD; Example Savings 1,200.00 EUR (= 1,302.44 USD); Cash 80.00 USD; Receivables 120.00 USD; Example Card −430.25 USD (limit 2,000.00 → available 1,569.75); Example Loan −5,000.00 USD.

Output (abridged): `{"as_of":"2026-11-03T15:10:00Z","reporting_currency":"USD","status":"completed","source_mode":"live","totals":{"assets":"3952.44","owed":"-5430.25","net_worth":"-1477.81","excluded_accounts":0},"by_currency":{"USD":"-2780.25","EUR":"1200.00"},"fx":{"snapshot_id":"fx_2026_11_03","as_of":"2026-11-03T00:00:00Z","source":"Primary rates provider","status":"fresh"},"flags":[],"limitations":[]}`

## E. Constraints and budgets

- **Hard constraints:** read-only — no writes to the external app, the hub or rates; never move money; the external app's live balances are the source of truth, and cached balances are always labelled cached with their as-of time; amounts are signed decimal strings with explicit currency; per-currency subtotals are always shown; a converted total only with non-expired, cross-checked (or labelled stale) FX and its source and as-of time; accounts without a rate or kind are flagged, never silently dropped; the API token stays server-side.
- **Soft constraints:** group by kind; show available credit for cards; keep the reply short with a link to details.
- **Resource and tenant scope:** the requesting user's accounts only.
- **Data sources allowed:** external app accounts, hub metadata and cached balances, FX snapshots.
- **External transfers permitted:** authenticated read request to the external app (no user data sent beyond the token).
- **Time limit:** 20 seconds per request.
- **Cost or token budget:** one external API call per request (cached for 5 minutes to respect provider limits); no LLM.
- **Maximum steps:** 9 (S1–S9).
- **Maximum retries and correction cycles:** external call 1 retry (honour Retry-After up to 10 s, else cached); reads 2; correction cycles 1.
- **Rate limits and concurrency:** respect the provider's limit (`TBD`; the design assumes a few hundred calls per hour); a 5-minute response cache per user.
- **Retention and logging constraints:** overviews are not stored by default; logs hold counts and flags only, never balances.
- **Prohibited actions:** presenting cached numbers as live; using expired rates for a total; hiding negative balances; exposing the token.
- **Stop conditions:** no external access and no cached balances (`failed`); kill switch.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | none (an optional overview snapshot only if the user asks to save one) | — | Not Allowed by default | — |
| Read | External accounts, hub metadata, cached balances, FX | own tenant | Allowed | — |
| Update | any | — | Not Allowed | — |
| Delete | any | — | Not Allowed | — |
| External side effect | authenticated GET to the external app | token valid | Allowed | response validation |

- **Idempotency:** read-only; the 5-minute response cache is keyed per user.
- **Expected side effects:** none beyond provider API usage.
- **Compensation:** not applicable — nothing changes.
- **Irreversible:** none.
- **Ambiguous outcomes:** a timed-out live call → cached mode with a label; no retry storm.

## G. Autonomy and approval policy

- **Default autonomy level:** A2 — bounded autonomy (read-only, owner-only output).
- **Planning level:** A2.
- **Execution level:** A2.
- **Verification level:** A2.
- **Recovery level:** A2 — labelled cached fallback only.
- **Approval required for:** connecting the external app (one-time, outside this recipe); saving an overview snapshot (if requested).
- **Actions never permitted:** any write; money movement; sharing the overview with anyone but the requesting user.
- **Conditions requiring clarification:** reporting currency not set.
- **Conditions requiring escalation:** the external app authorization expired (the user must reconnect); drift between live and cached balances above the threshold on several accounts (sync may be broken — tell the user).
- **Safe state after stop, cancellation or escalation:** nothing changed; partial results labelled.

## H. Execution plan

**S1 — Validate the request**
1. Objective: know the reporting currency and scope.
2. Capability: none (local validation).
3. Input mapping: request → `request@0.1.0`.
4. Dependencies: none.
5. Preconditions: user identified; kill switch off.
6. Action / side effects: validate; check the 5-minute cache (a hit returns the cached overview with its original `as_of`).
7. Autonomy: A2.
8. Timeout / retry: 2 s / none.
9. Idempotency: read-only.
10. Postconditions: valid request.
11. Verification: schema valid.
12. Failure routes: no reporting currency → ask once; else per-currency only.
13. Escalation triggers: none.
14. Evidence: request hash, cache hit or miss.

**S2 — Read hub account metadata**
1. Objective: kinds, limits and the mapping to external accounts.
2. Capability: `nova.finance.hub.accounts_list`.
3. Input mapping: `include_kinds`, active only.
4. Dependencies: S1.
5. Preconditions: read scope.
6. Action / side effects: read-only.
7. Autonomy: A2.
8. Timeout / retry: 5 s / 2.
9. Idempotency: read-only.
10. Postconditions: list of `hub-account` rows.
11. Verification: rows belong to the user; currencies are valid codes.
12. Failure routes: unavailable → continue with external data only; kinds become `unknown` (flagged).
13. Escalation: none.
14. Evidence: account count.

**S3 — Fetch live balances (or cached)**
1. Objective: current balances from the source of truth.
2. Capability: `nova.external_finance_app.accounts.list.live-rest`; fallback `nova.finance.hub.cached_balances.default`.
3. Input mapping: server-held token reference.
4. Dependencies: S1.
5. Preconditions: connection authorized.
6. Action / side effects: one authenticated GET; if it fails, is rate-limited beyond 10 s or reports a sync in progress, read cached balances instead (whole set or per missing account) and label them.
7. Autonomy: A2.
8. Timeout / retry: 8 s / 1 retry.
9. Idempotency: read-only.
10. Postconditions: a balance per account with `balance_source` and `as_of`.
11. Verification: HTTP 200 and parse for live; for cached, `as_of` present.
12. Failure routes: unauthorized → cached mode plus escalation; neither available → `failed`.
13. Escalation: expired authorization → ask the user to reconnect.
14. Evidence: source mode, sync signal, fetched_at.

**S4 — Validate balances and map accounts**
1. Objective: clean, typed inputs.
2. Capability: deterministic checks.
3. Input mapping: external accounts + hub metadata.
4. Dependencies: S2, S3.
5. Preconditions: balances present.
6. Action: parse balances as decimals; drop archived or inactive accounts (listed in limitations); map by external ID; external accounts without a hub mapping → kind `unknown`, flag `unmapped`; hub accounts missing from the external app → cached balance, flag `cached`; currency mismatch between hub and external → flag and use the external currency.
7. Autonomy: A2.
8. Timeout / retry: 1 s / none.
9. Idempotency: pure.
10. Postconditions: account rows ready for conversion.
11. Verification: every balance parses; every account has a currency.
12. Failure routes: an unparseable balance → exclude the account from totals, flag it, `partially_completed`.
13. Escalation: none.
14. Evidence: mapped, unmapped and excluded counts.

**S5 — Get exchange rates**
1. Objective: the latest verified snapshot.
2. Capability: `nova.finance.fx.get_rates.default`.
3. Input mapping: currencies present, `reporting_currency`.
4. Dependencies: S4.
5. Preconditions: more than one currency, or a reporting currency different from the account currency.
6. Action / side effects: read-only.
7. Autonomy: A2.
8. Timeout / retry: 3 s / 2.
9. Idempotency: read-only.
10. Postconditions: a snapshot with status, or `none`.
11. Verification: the snapshot covers the needed currencies; status computed.
12. Failure routes: missing currency → that account flagged `no_rate`; `expired`, `unverified` or `none` → no converted total.
13. Escalation: none (the FX recipe alerts on its own).
14. Evidence: snapshot ID and status.

**S6 — Convert and aggregate**
1. Objective: assets, owed, net worth and subtotals.
2. Capability: `nova.finance.networth.compute.default`.
3. Input mapping: account rows, snapshot, rules (Section D).
4. Dependencies: S5.
5. Preconditions: none.
6. Action: per-currency native subtotals; converted balances; totals (only if the FX status allows); available credit for cards and bnpl.
7. Autonomy: A2.
8. Timeout / retry: 1 s / none.
9. Idempotency: pure.
10. Postconditions: draft overview.
11. Verification: recomputed in S8.
12. Failure routes: none.
13. Escalation: none.
14. Evidence: computation inputs (IDs, not balances, in logs).

**S7 — Flag staleness and consistency**
1. Objective: make every weakness visible.
2. Capability: deterministic rules; `nova.finance.hub.cached_balances` for the drift check.
3. Input mapping: account rows, `stale_after_days`, sync signal, FX status.
4. Dependencies: S6.
5. Preconditions: none.
6. Action: flags `cached`, `provider_syncing`, `stale_balance` (balance as-of older than `stale_after_days`), `no_rate`, `unknown_kind`, `unmapped`; if `drift_check` and both live and cached values exist, flag `drift` where they differ (often records not yet synced); overview-level flags for FX `stale` and `mixed` source mode.
7. Autonomy: A2.
8. Timeout / retry: 5 s / 1.
9. Idempotency: read-only.
10. Postconditions: flags complete.
11. Verification: every cached balance carries the `cached` flag.
12. Failure routes: drift data unavailable → skip the drift check and say so.
13. Escalation: drift on 3 or more accounts → suggest checking the sync.
14. Evidence: flag counts.

**S8 — Independent recompute**
1. Objective: no wrong totals reach the user.
2. Capability: a separate verifier routine.
3. Input mapping: account rows + snapshot + draft overview.
4. Dependencies: S7.
5. Preconditions: none.
6. Action: recompute the totals from the per-currency subtotals (sum of each currency subtotal converted) and compare with the per-account sum; check that net = assets + owed.
7. Autonomy: A2.
8. Timeout / retry: 1 s / none.
9. Idempotency: pure.
10. Postconditions: verified or flagged.
11. Verification: both paths are equal to 0.01 in the reporting currency.
12. Failure routes: mismatch → `totals = null`, per-currency only, `partially_completed`, defect logged.
13. Escalation: none.
14. Evidence: verifier result.

**S9 — Deliver**
1. Objective: an honest, readable overview.
2. Capability: reply in the conversation or render on the dashboard.
3. Input mapping: `overview`.
4. Dependencies: S8.
5. Preconditions: requester is the account owner.
6. Action / side effects: for example, "Net worth −1,477.81 USD (live balances; rates fresh as of 3 Nov). Assets 3,952.44 · Owed 5,430.25 · Card available 1,569.75." Flags appear first when present ("2 balances are cached from 1 Nov"). Cache the overview for 5 minutes.
7. Autonomy: A2.
8. Timeout / retry: 5 s / 2.
9. Idempotency: per-request cache key.
10. Postconditions: delivered.
11. Verification: delivery status of the reply or page render (the page must render real content, not an error shell).
12. Failure routes: delivery fails → `partially_completed`.
13. Escalation: none.
14. Evidence: delivery status.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Read-only | zero writes anywhere | scope audit log | zero | stop; report defect |
| Source honesty | every cached balance labelled with its as-of time | overview | 100 % | fix labels before delivery |
| FX honesty | totals only with fresh or labelled stale FX | overview vs snapshot | holds | totals null |
| Completeness | every active account appears (included or flagged) | S4 counts | all accounted | add to limitations |
| Arithmetic | two-path recompute equal to 0.01 | S8 | equal | totals null |
| Sign | owed ≤ 0, assets ≥ 0, net = assets + owed | overview | holds | totals null |
| Token safety | token never in output or logs | output and log scan in tests | none | stop; rotate; escalate |
| Output schema | `overview` valid | validator | valid | fix |

## J. Self-correction and recovery

- **Failure classes:** configuration, external authorization, timeout or rate limit, provider syncing, unmapped account, missing rate, expired FX, arithmetic mismatch, delivery failure.
- **Permitted corrections:** labelled cached fallback; per-currency-only output; excluding unparseable accounts with a flag.
- **Forbidden corrections:** presenting cached numbers as live; using an expired or unverified rate for a total; guessing a missing balance; writing anything.
- **Maximum correction cycles:** 1 per step.
- **Reverification required after correction:** yes — S8.
- **Stop conditions:** no balance source; kill switch.
- **Recovery or compensation:** not applicable (read-only).
- **What must be reported:** source mode, every flag, excluded accounts, FX status.

## K. Independent challenge

- **Assumptions to challenge:** the external app's balance reflects today (a manual-entry app is only as current as its last entry); card balances are negative when owed (the convention is per provider — checked by kind and sign).
- **Alternative explanation:** a sudden net-worth drop may be a pending sync or a duplicate record, not real spending; drift flags point to this.
- **Contradictory evidence:** live vs cached drift; an account the hub lists but the app does not.
- **Could mislead:** a single net-worth number that hides stale inputs — flags are shown before the number; receivables tracked only in the person ledger (`nova.finance.lent-borrowed-ledger`) are not included unless mirrored as an account, and the overview says so.
- **Separate verifier:** S8 recomputes through a second path.
- **Findings requiring disclosure:** cached or mixed source, stale balances, FX status, excluded or unmapped accounts.

## L. Human escalation

- **Triggers:** external app authorization expired; drift on 3 or more accounts; repeated `failed`.
- **Why the agent cannot decide:** only the user can reconnect the app or investigate their records.
- **Information presented:** source mode, affected accounts (masked), last successful live fetch.
- **Decision requested:** reconnect, or accept cached figures.
- **Options and consequences:** reconnecting restores live balances; accepting keeps cached, labelled numbers.
- **Safe behaviour while waiting:** keep serving labelled cached overviews.
- **If no response:** continue in cached mode with the label.
- **Resume:** the next request after reconnecting uses live data.

## M. Completion conditions

Mandatory: `overview` valid; read-only held; every cached value labelled; FX rule held; totals verified or nulled; all active accounts included or flagged; evidence references present.

- `completed` — live balances for all accounts, FX fresh or labelled stale, totals verified, delivered.
- `partially_completed` — any cached balances, flagged or excluded accounts, totals nulled, or delivery failed.
- `blocked` — no reporting currency and the user declined to set one (per-currency output only is still returned), or the external app was never connected and no cached data exists.
- `failed` — no balance source available, or the read-only check failed.
- `cancelled` — kill switch or the user stopped the request.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome ("Net worth −1,477.81 USD, live, rates fresh"); `overview`; step statuses S1–S9; verification results; evidence references (FX snapshot ID, source mode, fetched_at); approvals and scope (read scopes); side effects (none) and reconciliation status; corrections (cached fallback used); unresolved challenges; limitations (excluded accounts, receivables not mirrored); escalation state; learning candidate. Never include the API token or full account numbers.

## O. Learning and revision

- **Outcome metrics:** live vs cached ratio, flags per request, drift frequency, unmapped accounts, FX-null rate, verifier mismatches, latency.
- **Evidence retained:** run records with counts and flags only.
- **Data minimisation:** no balances in logs; overviews not stored unless the user saves one.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}` — for example "provider reports card balances as positive → per-provider sign rule".
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (sandbox external app, fresh FX) | correct totals, live mode | UNVERIFIED on Skill Port; observed in the reference implementation |
| Invalid input (bad currency code) | asks; else per-currency only | TBD |
| Missing input (no reporting currency) | per-currency only, `partially_completed` | TBD |
| Missing capability (external binding absent) | cached mode, labelled | TBD |
| Permission denied (external token expired) | cached mode + reconnect prompt | TBD |
| Unauthorized scope (another tenant's account ref) | refused | TBD |
| Tenant isolation | only the user's accounts appear | TBD |
| Provider syncing signal | cached balances, `provider_syncing` flag | TBD |
| Expired FX | totals null; subtotals shown | TBD |
| Missing rate for one currency | account flagged `no_rate`; excluded from totals | TBD |
| Unmapped external account | included as `unknown`, flagged | TBD |
| Timeout (external call) | cached fallback within the time limit | TBD |
| Duplicate / replay (two requests in 5 minutes) | second served from cache with original as-of | TBD |
| False success (200 with an empty account list while the hub lists accounts) | treated as suspect; cached mode; flagged | TBD |
| Failed verification (injected arithmetic error) | totals null; `partially_completed` | TBD |
| Correction budget exhausted | safe stop | TBD |
| Escalation (drift on 3 accounts) | sync check suggested | TBD |
| Cancellation | `cancelled` | TBD |
| Rollback / compensation | not applicable — read-only; zero-write audit tested instead | TBD |
| Token safety | token absent from output and logs | TBD |
| Output schema and status | valid overview | TBD |
| Compatibility / regression (provider API version change) | contract test fails safely to cached mode | TBD |

All tests use a sandbox external-app account with fictional balances and recorded FX snapshots; never real accounts. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, the external app provider and its API version and rate limit — TBD (owner: publisher/operator).
- Account types differ by provider (some have no card or loan type); the hub's `kind` metadata carries the meaning, and must be maintained by the user.
- Investment and property values are only as good as the balances the external app holds.
- Receivables tracked only in the person ledger are excluded unless mirrored as an account.
- The privacy blur on shared screens is a UI concern outside this recipe.
