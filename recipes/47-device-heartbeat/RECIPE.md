# Device heartbeat

## A. Identity and ownership

- **Recipe ID:** `nova.ops.device-heartbeat` (proposed until assigned by Skill Port)
- **Name:** Device heartbeat
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** operations
- **Tags:** monitoring, devices, mesh-vpn, heartbeat, state-transitions, alert-once, read-only
- **Summary:** On a schedule, the agent reads the status of the user's devices on a private device network (for example a mesh VPN), treats a device that has been offline for more than 24 hours as late, and sends the owner exactly one alert when a device goes from ok to late and exactly one when it comes back. It never alerts on the first run, and it keeps state between runs.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** A private device network that exposes per-device status read-only (a mesh VPN client's local status command, or its control-plane API with a read-only key); a durable state store; an owner notification channel; optionally a jobs/health board. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — the behaviour runs in Nova's reference implementation (status read from the mesh VPN client's JSON status output every 5 minutes, the 24-hour late rule, once-per-transition alerts, no first-run alert, a state file between runs, and a red/green jobs-board badge). Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.devices.status` | Capability | Read the device network's status without changing it | low | UNVERIFIED |
| `nova.devices.status.list` | Operation (read) | Return each device's ID, display name, online flag and last-seen time | low | UNVERIFIED |
| `nova.devices.heartbeat` | Capability | Classify devices and compute state transitions | low | UNVERIFIED |
| `nova.devices.heartbeat.evaluate` | Operation (read, pure function) | Map current status + previous state → new state + transitions | low | UNVERIFIED |
| `nova.devices.heartbeat_state` | Capability | Keep per-device heartbeat state between runs | low | UNVERIFIED |
| `nova.devices.heartbeat_state.get` | Operation (read) | Load the previous state and its version | low | UNVERIFIED |
| `nova.devices.heartbeat_state.put` | Operation (create on first run, then update) | Save the new state atomically with a version check | low | UNVERIFIED |
| `nova.jobs.registry` | Capability | Show device health on the operator's jobs/health board | low | UNVERIFIED |
| `nova.jobs.registry.report` | Operation (update) | Upsert this recipe's device rows (state, last seen, run time) | low | UNVERIFIED |
| `nova.notify.owner` | Capability | Deliver a message to the requesting user's own channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Send a transition alert to the owner | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.devices.status.list.local-cli` — the mesh VPN client's local status command on the host that runs the agent (machine-readable JSON output). Needs no API credential; read-only. Selected by default.
- `nova.devices.status.list.control-api` — the mesh VPN control-plane REST API, called with a key scoped to reading devices only. Eligible only when the operator has configured that key. Fallback from `local-cli` to `control-api` is allowed only when both are configured, because both are read-only and return the same normalised schema; the switch is recorded in the run record. No variant that can authorise, expire, rename or remove devices is declared, so no fallback to one is possible.
- `nova.devices.heartbeat_state.get|put.file` — one JSON file with mode `0600`, written by temp file → fsync → atomic rename, guarded by a run lock and a `version` compare-and-swap.
- `nova.devices.heartbeat_state.get|put.database` — one row per tenant in a database table, written only through a security-definer function that enforces an optimistic `version` check.
  The configured store is used for both `get` and `put`. There is **no fallback between stores**: switching stores mid-life would lose state and could cause duplicate or missed alerts, so a store change is an operator action that rebaselines (Section G).
- `nova.devices.heartbeat.evaluate.default`, `nova.jobs.registry.report.default`, `nova.notify.owner.send.default` — only the default variant is supported.

**Bindings:** `binding://nova/devices/status-local`, `binding://nova/devices/status-api`, `binding://nova/devices/heartbeat-evaluate`, `binding://nova/devices/state-file`, `binding://nova/devices/state-db`, `binding://nova/jobs/report`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Reference implementation: a collector script on a systemd timer that reads the mesh VPN client's JSON status, keeps a state file in a service-owned directory, writes device rows into a jobs registry through a database dispatcher function, and sends owner alerts to a chat channel.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Device status record | `schema://nova/device-heartbeat/device-status@0.1.0` | the user's own device network | read |
| Heartbeat state | `schema://nova/device-heartbeat/heartbeat-state@0.1.0` | the user's tenant | create (first run), read, update |
| Transition | `schema://nova/device-heartbeat/transition@0.1.0` | inside one heartbeat state | create |
| Job row | `schema://nova/device-heartbeat/job-report@0.1.0` | the operator's jobs board, this recipe's rows only | create, update |
| Owner alert | `schema://nova/device-heartbeat/alert@0.1.0` | the owner channel | create |

**Authorization requirements (declared, never a grant):** `devices:status:read`, `heartbeat:state:read`, `heartbeat:state:write`, `jobs:report`, `notify:owner`. Credentials live in the runtime only: `secret_ref://mesh-vpn-readonly-key` (control-api variant), `secret_ref://state-store` (database variant), `secret_ref://owner-channel`.

**Policy / trust:** risk class `low` — read-only monitoring with owner-only messages. Data minimisation applies at the boundary: network addresses, endpoints, keys, operating-system details and account identities in the status output are discarded in S3 and never stored, logged or sent.

## C. Intent and applicability

- **User problem:** The user relies on always-on machines (a home server, a desktop that runs local automations, a small single-board computer) and only notices one has dropped off when something downstream fails days later.
- **Desired outcome:** A quiet monitor that speaks only when a device has really been gone for a day, and once more when it is back — never a stream of repeats, and never a false "all good" when it cannot see.
- **Trigger or entry conditions:** A scheduled run (default every 5 minutes) once the operator has enabled the recipe. A manual "check my devices now" request runs the same path and is serialised by the run lock.
- **Applicable situations:** Servers, desktops and single-board computers that the user owns and has joined to their private device network.
- **Out-of-scope situations:** Any device-management action (authorise, expire, rename, remove, change access rules); devices the user does not own; tracking people or locations; inspecting network traffic; uptime commitments to third parties; detecting short outages (use a dedicated uptime monitor for sub-hour detection).
- **Required inputs:** `config` (see Section D).
- **Prerequisites:** The device network client is installed and logged in on the host (local-cli variant), or a read-only control-plane key is stored in the runtime (control-api variant); a durable state store; an owner channel; NTP-synchronised clocks.
- **Required capabilities:** `nova.devices.status`, `nova.devices.heartbeat`, `nova.devices.heartbeat_state`, `nova.notify.owner`; optional `nova.jobs.registry` (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port; every binding is a dependency to resolve before activation.
- **Assumptions:** Verified in the reference implementation: per-device online flag and last-seen time from the mesh VPN client, the 24-hour rule, one alert per transition, no first-run alert, persistent state. Assumed: the target network reports a last-seen time for offline devices.
- **Ambiguity policy:** Use a documented safe default. Selectors that match nothing are ignored with a warning in the run record; two devices with the same display name are tracked by device ID, never merged. If the watch list resolves to zero devices, stop (`blocked`) and tell the owner once.
- **Vedaxi task type:** Not applicable — this recipe does not map to any of the seven task types Vedaxi supports today.

## D. Inputs and outputs

**Inputs** (operator configuration, captured once at enablement)

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `watch` | array of selectors: a device ID or `tag:<name>` | no | operator; low | all devices carrying `tag:watch` | none resolve → `blocked`, owner told once |
| `late_after_hours` | integer 1–720 | no | operator | 24 | out of range → clamp and disclose |
| `retired` | array of device IDs | no | operator | `[]` | unknown ID → ignore with a warning |
| `source_variant` | enum `local-cli`, `control-api` | no | operator | `local-cli` | `control-api` without a key → `blocked` |
| `interval_minutes` | integer 5–60 | no | operator | 5 | out of range → clamp |
| `source_alert_after_runs` | integer 2–20 | no | operator | 3 | out of range → clamp |

```json
{"$id":"schema://nova/device-heartbeat/config@0.1.0","type":"object","additionalProperties":false,
 "properties":{"watch":{"type":"array","items":{"type":"string","pattern":"^(tag:[a-z0-9-]{1,40}|[A-Za-z0-9_-]{1,64})$"},"maxItems":200},
 "late_after_hours":{"type":"integer","minimum":1,"maximum":720,"default":24},
 "retired":{"type":"array","items":{"type":"string"},"maxItems":200},
 "source_variant":{"enum":["local-cli","control-api"],"default":"local-cli"},
 "interval_minutes":{"type":"integer","minimum":5,"maximum":60,"default":5},
 "source_alert_after_runs":{"type":"integer","minimum":2,"maximum":20,"default":3}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `heartbeat_run` | object `schema://nova/device-heartbeat/heartbeat-run@0.1.0` | `run_id`, `status`, `observed_at`, `devices`, `transitions`, `alerts_sent`, `first_run` | run record; summary on the jobs board | low (device names only) | Section I |
| owner alerts | `schema://nova/device-heartbeat/alert@0.1.0` | `idempotency_key`, `device_id`, `transition`, `text` | owner channel only | low | delivery receipt |

```json
{"$id":"schema://nova/device-heartbeat/heartbeat-run@0.1.0","type":"object",
 "required":["run_id","status","observed_at","devices","transitions","alerts_sent","first_run"],
 "properties":{"run_id":{"type":"string"},"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},
 "observed_at":{"type":"string","format":"date-time"},"source_variant":{"enum":["local-cli","control-api"]},"first_run":{"type":"boolean"},
 "devices":{"type":"array","items":{"type":"object","required":["device_id","state"],"properties":{"device_id":{"type":"string"},"name":{"type":"string"},
  "state":{"enum":["ok","late","unknown","retired"]},"last_seen":{"type":["string","null"],"format":"date-time"},"offline_hours":{"type":["number","null"]}}}},
 "transitions":{"type":"array","items":{"$ref":"schema://nova/device-heartbeat/transition@0.1.0"}},
 "alerts_sent":{"type":"integer","minimum":0},"alerts_pending":{"type":"integer","minimum":0},
 "warnings":{"type":"array","items":{"type":"string"}},"state_version":{"type":"integer","minimum":1}}}
```

**Other schemas** (compact JSON Schema):

```json
{"$id":"schema://nova/device-heartbeat/status-query@0.1.0","type":"object","required":["source_variant"],"properties":{"source_variant":{"enum":["local-cli","control-api"]},"selectors":{"type":"array","items":{"type":"string"}}}}
{"$id":"schema://nova/device-heartbeat/device-status@0.1.0","type":"object","required":["device_id","online","observed_at"],"additionalProperties":false,"properties":{"device_id":{"type":"string"},"name":{"type":"string","maxLength":64},"online":{"type":"boolean"},"last_seen":{"type":["string","null"],"format":"date-time"},"tags":{"type":"array","items":{"type":"string"}},"observed_at":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/device-heartbeat/status-list@0.1.0","type":"object","required":["observed_at","source_variant","devices"],"properties":{"observed_at":{"type":"string","format":"date-time"},"source_variant":{"enum":["local-cli","control-api"]},"devices":{"type":"array","items":{"$ref":"schema://nova/device-heartbeat/device-status@0.1.0"}}}}
{"$id":"schema://nova/device-heartbeat/state-key@0.1.0","type":"object","required":["tenant_id","store"],"properties":{"tenant_id":{"type":"string"},"store":{"enum":["file","database"]}}}
{"$id":"schema://nova/device-heartbeat/heartbeat-state@0.1.0","type":"object","required":["version","updated_at","devices","source_failures"],"properties":{"version":{"type":"integer","minimum":1},"updated_at":{"type":"string","format":"date-time"},"source_failures":{"type":"integer","minimum":0},"source_alerted":{"type":"boolean"},"devices":{"type":"object","additionalProperties":{"type":"object","required":["state","since"],"properties":{"state":{"enum":["ok","late"]},"since":{"type":"string","format":"date-time"},"last_seen":{"type":["string","null"],"format":"date-time"},"pending_alert":{"type":["object","null"],"properties":{"idempotency_key":{"type":"string"},"transition":{"enum":["ok_to_late","late_to_ok"]},"attempts":{"type":"integer"}}}}}}}}
{"$id":"schema://nova/device-heartbeat/transition@0.1.0","type":"object","required":["device_id","from","to","at","idempotency_key"],"properties":{"device_id":{"type":"string"},"from":{"enum":["ok","late"]},"to":{"enum":["ok","late"]},"at":{"type":"string","format":"date-time"},"offline_hours":{"type":"number"},"idempotency_key":{"type":"string","pattern":"^[a-f0-9]{64}$"}}}
{"$id":"schema://nova/device-heartbeat/alert@0.1.0","type":"object","required":["idempotency_key","device_id","transition","text"],"properties":{"idempotency_key":{"type":"string"},"device_id":{"type":"string"},"transition":{"enum":["ok_to_late","late_to_ok","source_unavailable","source_restored","state_reset"]},"text":{"type":"string","maxLength":500}}}
{"$id":"schema://nova/device-heartbeat/delivery@0.1.0","type":"object","required":["idempotency_key","delivered"],"properties":{"idempotency_key":{"type":"string"},"delivered":{"type":"boolean"},"message_id":{"type":["string","null"]},"error":{"type":["string","null"]}}}
{"$id":"schema://nova/device-heartbeat/job-report@0.1.0","type":"object","required":["job_key","lane","state","reported_at"],"properties":{"job_key":{"type":"string","pattern":"^device:[A-Za-z0-9_-]+$"},"lane":{"const":"device"},"state":{"enum":["ok","late","unknown","disabled"]},"last_seen":{"type":["string","null"],"format":"date-time"},"reported_at":{"type":"string","format":"date-time"}}}
```

Notes: `device-status@0.1.0`: `additionalProperties:false` is what keeps addresses, keys and account identities out.

**Example (fictional)**

Input (config): `{"watch":["tag:watch"],"late_after_hours":24,"source_variant":"local-cli"}`

Previous state: `studio-desktop` is `ok`, last seen `2026-10-30T08:12:00Z`. Status now (`2026-10-31T09:20:00Z`): `studio-desktop` offline, `home-server` online.

Output: `{"run_id":"hb_0412","status":"completed","observed_at":"2026-10-31T09:20:00Z","source_variant":"local-cli","first_run":false,"devices":[{"device_id":"dev_a1","name":"studio-desktop","state":"late","last_seen":"2026-10-30T08:12:00Z","offline_hours":25.1},{"device_id":"dev_b2","name":"home-server","state":"ok","last_seen":"2026-10-31T09:20:00Z","offline_hours":0}],"transitions":[{"device_id":"dev_a1","from":"ok","to":"late","at":"2026-10-31T09:20:00Z","offline_hours":25.1,"idempotency_key":"<sha256>"}],"alerts_sent":1,"alerts_pending":0,"state_version":418}`

Alert text: "studio-desktop has been offline for 25 h (last seen 30 Oct 08:12 UTC). Late threshold: 24 h."

## E. Constraints and budgets

- **Hard constraints:** read-only towards the device network — no device-management call of any kind; alerts go to the owner channel only; no alert on the first run or on a device's first observation; at most one alert per transition per device (`ok→late`, `late→ok`); `unknown` never triggers a device alert and is never shown as ok; a stale collector is never shown as healthy; network addresses, endpoints, keys, OS details and account identities are never stored, logged or sent; state writes are atomic with a version check; one run at a time per tenant.
- **Soft constraints:** short alerts that state the offline duration; several transitions in one run may share one message, but each transition is announced exactly once.
- **Resource and tenant scope:** the user's own device network and this recipe's own state and job rows.
- **Data sources allowed:** device-network status only.
- **External transfers permitted:** alert text to the configured owner-channel provider.
- **Time limit:** status read 15 s; whole run ≤ 60 s.
- **Cost or token budget:** no LLM is used; cost is negligible. Control-plane API rate limits apply to the control-api variant.
- **Maximum steps:** 8 (S1–S8).
- **Maximum retries and correction cycles:** status read: 2 retries per run; alert delivery: 2 retries per run, and an undelivered alert stays pending for at most 12 further runs before escalation; state write: 1 retry on version conflict; correction cycles: 1 per run.
- **Rate limits and concurrency:** 1 concurrent run per tenant; at most 1 control-plane API call per run.
- **Retention and logging constraints:** state holds only the current per-device state and any pending alert; run records kept 30 days (operator may change; `TBD` in policy); logs never contain raw status payloads.
- **Prohibited actions:** device management; reading traffic or device contents; alerting anyone other than the owner; changing the threshold mid-run; silently dropping a pending alert.
- **Stop conditions:** run lock already held; zero watched devices; source variant misconfigured; state version conflict unresolved after one retry; state store unreadable.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | Heartbeat state (version 1) | no prior state exists | Allowed | read-back returns version 1 |
| Create | Owner alert | a transition or source notice is pending | Allowed (owner only; scope approved at enablement) | delivery receipt |
| Read | Device status; heartbeat state | own network and tenant | Allowed | schema-valid |
| Update | Heartbeat state: per-device `state`, `since`, `last_seen`, `pending_alert`; `source_failures`, `source_alerted` | stored `version` equals the version read in S4 | Allowed | version incremented by 1 |
| Update | Jobs board: this recipe's `device:*` rows | rows owned by this recipe | Allowed | `reported_at` equals the run time |
| Delete | devices, state history, job rows | — | Not Allowed (a retired device's row is set to `disabled`, not deleted) | — |
| External side effect | the device network | — | Not Allowed (read-only) | no mutating call in the binding log |
| External side effect | message to the owner | pending alert in persisted state | Allowed | delivery receipt |

- **Idempotency:** alert key = sha256(`tenant`, `device_id`, `from`, `to`, `since`). The key is written into state (outbox) before sending; S7 skips keys already marked sent. State `version` compare-and-swap stops two runs from both deciding the same transition.
- **Expected side effects:** the owner receives a message only when a device changes between ok and late, or when the monitor itself cannot see (source notice).
- **Compensation:** a sent message cannot be unsent; a wrong alert is followed by one correction message that cites its key.
- **Irreversible:** delivered messages.
- **Ambiguous outcomes:** if delivery times out, the alert stays pending with `attempts` incremented. The next attempt reuses the same key; where the channel supports idempotent send or delivery lookup, it is used. Otherwise one resend is allowed and the run discloses a possible duplicate.

## G. Autonomy and approval policy

- **Default autonomy level:** A2 — bounded autonomy (read-only reads, owner-only messages).
- **Planning level:** A2.
- **Execution level:** A2 within the scope approved at enablement.
- **Verification level:** A2.
- **Recovery level:** A2 for bounded retries; A0 for a state reset or a store change.
- **Approval required for:** enabling the recipe (the operator approves the watch list, threshold, channel and store — a one-time scope); switching to the control-api variant (new credential); resetting or migrating state; adding any channel other than the owner's.
- **Actions never permitted:** device management; alerting third parties; suppressing a pending alert to tidy up; reporting `unknown` as ok; storing network addresses or keys.
- **Conditions requiring clarification:** zero devices resolved; a selector that matches devices the operator did not expect (shown in the run record for the operator to confirm).
- **Conditions requiring escalation:** status source unavailable for `source_alert_after_runs` consecutive runs (one notice); state corrupt; a pending alert still undelivered after 12 runs.
- **Safe state after stop, cancellation or escalation:** no state change from the stopped run, no message beyond those already delivered, the run recorded as `blocked`, `failed` or `cancelled`, and the next scheduled run proceeds normally.

Approval at enablement covers recurring owner-only alerts for the approved devices and channel. It does not cover new recipients, new channels, credentials or device-management actions.

## H. Execution plan

**S1 — Acquire the run lock and load configuration**
1. Objective: guarantee a single run and a valid configuration.
2. Capability: runtime lock (UNVERIFIED binding); local validation.
3. Input mapping: stored operator config → `config@0.1.0`.
4. Dependencies: none.
5. Preconditions: the recipe is enabled for this tenant.
6. Action / side effects: take a non-blocking lock; validate and clamp config; no external side effects.
7. Autonomy: A2; no approval.
8. Timeout / retry: 2 s / none (a held lock means another run is active).
9. Idempotency: not applicable.
10. Postconditions: lock held; schema-valid config.
11. Verification: JSON Schema validation passes; clamped values listed in `warnings`.
12. Failure routes: lock held → end the run as `cancelled` with reason `concurrent_run`; invalid config → `blocked`.
13. Escalation triggers: config invalid on 3 consecutive runs → tell the operator once.
14. Evidence: lock acquisition time, validated config hash.

**S2 — Read device status**
1. Objective: obtain the current status of the watched devices.
2. Capability: `nova.devices.status.list` (variant `local-cli`, or `control-api` when configured).
3. Input mapping: `source_variant`, `watch` → `status-query@0.1.0`.
4. Dependencies: S1.
5. Preconditions: the variant's prerequisites are met (client logged in, or key present by reference).
6. Action / side effects: one read-only call; no side effects on the network.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 2 retries on transient errors with backoff; the declared read-only fallback is allowed only when both variants are configured.
9. Idempotency: read-only.
10. Postconditions: a raw status response, or a recorded source failure.
11. Verification: the response parses; it contains a timestamp or the run supplies `observed_at` from the runtime clock.
12. Failure routes: all attempts fail → skip to S6 with `source_failures + 1` and no device state changes; permission error → `blocked`.
13. Escalation triggers: `source_failures` reaches `source_alert_after_runs` → one `source_unavailable` notice (never on the first run).
14. Evidence: variant used, latency, error class (not the payload).

**S3 — Normalise and minimise**
1. Objective: a clean, minimal, UTC status list.
2. Capability: local transformation.
3. Input mapping: raw response → `status-list@0.1.0`.
4. Dependencies: S2.
5. Preconditions: a parsed response.
6. Action / side effects: keep only device ID, display name, online flag, last-seen time and tags; convert every timestamp to UTC (handle naive-UTC strings and offset strings alike); drop addresses, endpoints, keys, OS and account fields; resolve `watch` selectors; mark `retired` devices.
7. Autonomy: A2.
8. Timeout / retry: 2 s / none.
9. Idempotency: deterministic.
10. Postconditions: every device record validates against `device-status@0.1.0` (`additionalProperties:false`).
11. Verification: schema validation; no timestamp more than 5 minutes in the future (clock skew is clamped to `observed_at` and flagged).
12. Failure routes: an unparseable timestamp → that device becomes `unknown` for this run; selectors resolving to zero devices → `blocked`.
13. Escalation triggers: zero devices resolved → tell the owner once.
14. Evidence: device count, selectors resolved, skew flags.

**S4 — Load previous state**
1. Objective: know each device's last confirmed state.
2. Capability: `nova.devices.heartbeat_state.get` (configured store).
3. Input mapping: tenant → state key.
4. Dependencies: S1.
5. Preconditions: store reachable.
6. Action / side effects: read-only.
7. Autonomy: A2.
8. Timeout / retry: 5 s / 1 retry.
9. Idempotency: read-only.
10. Postconditions: `heartbeat-state@0.1.0` with its `version`, or "no state" (first run).
11. Verification: schema validation.
12. Failure routes: store unreadable → `failed`, no messages; state fails validation (corrupt) → quarantine the copy, treat as first run (baseline) and queue one `state_reset` notice.
13. Escalation triggers: corrupt state → the `state_reset` notice tells the owner that transitions since the last good run may have been missed.
14. Evidence: state version read, or `first_run=true`, or the quarantine reference.

**S5 — Evaluate states and transitions**
1. Objective: decide each device's state and any transition, without false alarms.
2. Capability: `nova.devices.heartbeat.evaluate`.
3. Input mapping: `status-list`, previous state, `late_after_hours`, `observed_at`.
4. Dependencies: S3, S4.
5. Preconditions: both inputs valid.
6. Action / side effects: pure function. Rules: online → `ok`; offline with `observed_at − last_seen > late_after_hours` → `late`; offline within the threshold → `ok`; missing from the response or no usable `last_seen` → `unknown` (the stored state is kept unchanged). A transition is emitted only for `ok→late` or `late→ok` against the stored state. No transition is emitted when there is no prior state (first run) or the device is new to the state (first observation) — those are baselined. `unknown` is compared through the stored state, so `ok→unknown→late` still yields exactly one `ok→late`. Source recovery after a source notice yields one `source_restored` notice.
7. Autonomy: A2.
8. Timeout / retry: 2 s / none.
9. Idempotency: deterministic for the same inputs.
10. Postconditions: new candidate state; a list of transitions, each with an idempotency key.
11. Verification: invariants — a transition's `from` equals the stored state; no transition for first-run or first-observation devices; no transition involves `unknown`.
12. Failure routes: an invariant fails → discard the candidate, `failed`, no messages.
13. Escalation triggers: none.
14. Evidence: per-device decision with the rule that fired.

**S6 — Persist the new state (outbox)**
1. Objective: record decisions and pending alerts before any message is sent.
2. Capability: `nova.devices.heartbeat_state.put` (configured store).
3. Input mapping: candidate state with `pending_alert` set for each transition; `version` from S4.
4. Dependencies: S5 (or S2 on a source failure).
5. Preconditions: the version read in S4 is still current.
6. Action / side effects: atomic write (temp file + fsync + rename, mode `0600`; or a database compare-and-swap).
7. Autonomy: A2.
8. Timeout / retry: 5 s / 1 retry after re-reading on a version conflict (re-run S5 with the fresh state).
9. Idempotency: version compare-and-swap; the same transition key is never written twice.
10. Postconditions: stored `version` = previous + 1; pending alerts recorded.
11. Verification: read back the version and the pending keys.
12. Failure routes: conflict persists → `failed`, nothing sent (the next run decides again); write error → `failed`.
13. Escalation triggers: 3 consecutive write failures → tell the operator once.
14. Evidence: new version, pending keys.

**S7 — Deliver alerts**
1. Objective: tell the owner about each pending transition exactly once.
2. Capability: `nova.notify.owner.send`.
3. Input mapping: pending alerts from state → `alert@0.1.0` (device name, transition, offline duration, last-seen time in UTC and in the owner's time zone).
4. Dependencies: S6.
5. Preconditions: owner channel configured; the alert key is not already marked sent.
6. Action / side effects: send to the owner only; on success clear `pending_alert` with a second compare-and-swap write.
7. Autonomy: A2, within the enablement scope.
8. Timeout / retry: 10 s / 2 retries per run; an unsent alert stays pending for up to 12 further runs.
9. Idempotency: key = sha256(`tenant`, `device_id`, `from`, `to`, `since`); sent keys are never resent.
10. Postconditions: each delivered alert has a message ID; pending count reported.
11. Verification: delivery receipt per key; the state shows the key cleared.
12. Failure routes: delivery fails → keep pending, run `partially_completed`; clearing write fails → the next run sees the key pending but checks delivery by key before resending.
13. Escalation triggers: an alert still undelivered after 12 runs → record in the run and on the jobs board as `alert_undelivered`.
14. Evidence: message IDs per key, attempt counts.

**S8 — Report to the jobs board and close the run**
1. Objective: an honest visible status and a clean run record.
2. Capability: `nova.jobs.registry.report` (optional); run record.
3. Input mapping: per-device state → `job-report@0.1.0` rows `device:<id>`, lane `device`.
4. Dependencies: S6 (S7 if alerts were due).
5. Preconditions: board configured (otherwise skipped and noted).
6. Action / side effects: upsert this recipe's rows; `late` shows red; `retired` shows `disabled`; release the lock.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 1 retry.
9. Idempotency: upsert by `job_key`.
10. Postconditions: rows updated with `reported_at` = run time; `heartbeat_run` emitted; lock released.
11. Verification: board read-back; a board that has not received a report within 4 × `interval_minutes` must render `unknown`, never green (board-side rule, declared as a dependency).
12. Failure routes: board write fails → run `partially_completed`; alerts are unaffected.
13. Escalation triggers: none.
14. Evidence: board row timestamps, final `heartbeat_run`.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Input validity | config matches `config@0.1.0` | validator | valid after clamping | `blocked` |
| Read-only | no mutating call was made to the device network | binding call log | zero non-read calls | stop, `failed`, report a policy breach |
| Minimisation | stored and sent data contain no address, key or account field | `device-status` schema, alert text scan | none found | discard, `failed` |
| Time normalisation | all timestamps UTC; none > 5 min in the future | S3 output | all pass | clamp and flag |
| First-run silence | no alert when `first_run=true` or on a first observation | S5 decisions | zero alerts | discard candidate |
| Once per transition | each key is sent at most once | state outbox, message log | no duplicate key | disclose and correct |
| Unknown handling | `unknown` never produces a device alert or an ok badge | S5 decisions, board rows | holds | fix state, `failed` |
| State integrity | version incremented by exactly 1 per write | store read-back | holds | re-read and re-evaluate once |
| Freshness | board shows `unknown` when the last report is older than 4 × interval | board | holds | flag board dependency |
| Output schema | `heartbeat_run` valid | validator | valid | fix fields; never fake values |

A successful status read proves only that the source answered; it does not prove that every device is healthy.

## J. Self-correction and recovery

- **Failure classes:** validation, missing capability, permission denial, source timeout, transient source error, state conflict, corrupt state, delivery failure, failed invariant.
- **Permitted corrections:** clamp out-of-range config; retry reads (bounded); re-read state and re-evaluate once on a version conflict; carry an undelivered alert to later runs; quarantine corrupt state and rebaseline with one notice.
- **Forbidden corrections:** treat a source failure as "all ok"; mark `unknown` devices ok; drop pending alerts; widen the read scope or add a mutating variant; alert anyone other than the owner.
- **Maximum correction cycles:** 1 per run.
- **Reverification required after correction:** yes — re-run S5 invariants and S6 read-back.
- **Stop conditions:** correction budget used; hard constraint fails.
- **Recovery or compensation procedure:** a wrong alert is followed by one correction message citing its key.
- **What must be reported:** clamps, fallback variant use, source failures, pending or duplicate alerts, state resets.

## K. Independent challenge

- **Assumptions to challenge:** "offline" really means unreachable (not merely a renamed or re-keyed device that now appears under a new ID); `last_seen` is in UTC; the host's own clock is correct.
- **Alternative explanation:** the monitoring host itself lost the network — every device looks offline at once. If more than half the watched devices go late in the same run, hold the alerts for one more run and add a "monitor host may be offline" note.
- **Contradictory evidence:** a device reported offline whose jobs on the board are still succeeding.
- **Could mislead:** a green board while the collector has stopped; a "back online" alert caused by a duplicate device entry.
- **Separate verifier:** a rule check, independent of S5, recomputes transitions from the raw normalised list and the stored state and compares them before S6.
- **Findings requiring disclosure:** mass-offline hold, duplicate device IDs for one name, clock skew.

## L. Human escalation

- **Triggers:** status source unavailable for N runs; corrupt state; alert undeliverable after 12 runs; zero devices resolved; the mass-offline hold.
- **Why the agent cannot decide:** whether a device is retired, whether to re-credential the source, and whether to reset state are owner decisions.
- **Information presented:** affected devices (names only), last-seen times, failure class, the number of runs affected.
- **Decision requested:** fix the source; mark a device retired; approve a state reset; change the threshold.
- **Options and consequences:** retiring stops alerts for that device; resetting state rebaselines (no alert for transitions during the reset).
- **Safe behaviour while waiting:** keep running read-only; no new alerts for the same condition.
- **If no response:** nothing changes; the board stays red or `unknown` honestly.
- **Resume:** configuration changes take effect at the next run after S1 validation.

## M. Completion conditions

Mandatory: `heartbeat_run` valid; no mutating call to the device network; no first-run or first-observation alert; every transition announced at most once; `unknown` never reported as ok; minimisation check passed; evidence references present; limitations disclosed.

- `completed` — status read, states evaluated and persisted, every due alert delivered, board updated (or not configured).
- `partially_completed` — states persisted but an alert is still pending, or the board update failed.
- `blocked` — configuration, permission or zero-device condition prevented evaluation.
- `failed` — source unreadable, state store unreadable, version conflict unresolved, or an invariant failed; no state change.
- `cancelled` — another run held the lock, or the operator disabled the recipe before S6.

A run where the source could not be read is never `completed`, even though no device changed state.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome ("1 device late, 0 recovered"); `heartbeat_run`; step statuses S1–S8; verification results and failed checks; evidence references (state version, alert keys, message IDs); approvals and scope (the enablement scope); side effects (messages sent) and reconciliation status (pending alerts); corrections performed; unresolved challenges (mass-offline hold); limitations; escalation state; learning candidate. Never include network addresses, keys, raw status payloads or account identities.

## O. Learning and revision

- **Outcome metrics:** alerts per device per month, mean time from last seen to alert, source-failure rate, pending-alert age, false-alert reports from the owner.
- **Evidence retained:** run records and transition keys, per retention policy.
- **Data minimisation:** metrics keep device IDs and counts only; no status payloads.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}`.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review; a revision cannot add a mutating variant or a new recipient without a new enablement approval.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (fixture: one device ok → offline 25 h) | one `ok→late` alert; state version +1 | UNVERIFIED on Skill Port; observed in the reference implementation |
| First run with a device already offline 3 days | baselined as `late`, zero alerts | UNVERIFIED on Skill Port; observed in the reference implementation |
| Still late on the next run | no second alert | TBD |
| Back online | exactly one `late→ok` alert | TBD |
| Offline 23 h then back | no alert at all | TBD |
| New device appears after baseline | baselined, no alert | TBD |
| Missing input (no selectors match) | `blocked`, owner told once | TBD |
| Invalid input (`late_after_hours: 0`) | clamped to 1, disclosed | TBD |
| Missing capability (status binding absent) | `blocked` with a dependency message | TBD |
| Permission denied (`devices:status:read` missing) | `blocked`, no state change | TBD |
| Unauthorized scope (key with write scope supplied) | rejected at enablement — read-only key required | TBD |
| Tenant isolation | cannot read another tenant's state or rows | TBD |
| Source timeout ×3 runs | one `source_unavailable` notice; later one `source_restored` | TBD |
| Duplicate / replay (two runs at once) | lock or version CAS lets one decide; one alert | TBD |
| Delivery timeout | alert stays pending; resent with the same key; no duplicate where the channel dedupes | TBD |
| False success (source answers with an empty list) | devices `unknown`; no ok badge; no alert | TBD |
| Failed verification (S5 invariant broken by fixture) | candidate discarded, `failed` | TBD |
| Corrupt state file | quarantined, rebaselined, one `state_reset` notice | TBD |
| Time-zone normalisation (naive UTC vs offset strings) | identical decisions | TBD |
| Mass offline (monitor host loses network) | alerts held one run with a note | TBD |
| Correction budget exhausted (conflict twice) | `failed`, nothing sent | TBD |
| Escalation (alert undeliverable 12 runs) | recorded as `alert_undelivered` | TBD |
| Cancellation (recipe disabled before S6) | `cancelled`, no state change | TBD |
| Compensation (wrong alert) | one correction message citing the key | TBD |
| Output schema and status | valid `heartbeat_run`, correct status | TBD |
| Compatibility / regression (new client JSON format) | normaliser still yields valid `device-status` | TBD |

Fixtures use fictional devices on a test network; no real devices are needed. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs, schema registry and the jobs-board contract — TBD (owner: publisher, with Skill Port).
- Not every device network reports `last_seen` for offline devices; without it devices stay `unknown` (owner: publisher, per network adapter).
- Detection granularity is the 24-hour threshold plus one interval; short outages are not detected by design.
- Run-record retention period — TBD (owner: operator).
- The mass-offline hold threshold (more than half of watched devices) is a design default, not a measured value.
- Whether to support a second owner-approved recipient (for example a household member) — open; would need a new enablement approval.
