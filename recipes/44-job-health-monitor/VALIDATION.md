# Validation report — Job health monitor (`nova.ops.job-health-monitor` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`). The manifest uses a templated binding reference `binding://nova/jobs/source-{variant}`; each variant's concrete binding is listed in its constraints.
- Elapsed-time tolerance for sources without a next-run time; failure window for long-cadence jobs; retention of cycle and alert records (Sections B, E, Q).
- `identity_provider` for the collector service identity and readers (`TBD`).

## Contradictions

None found. Checked: "stale ≠ healthy" vs a failing source (rows are not refreshed and turn `unknown`); "failing only on outages" vs daily jobs (one failed daily run with no success in the window is `failing` by R3); "judge timers by next-run time" vs the collector measuring itself mid-run (executing service counts as `running`); "on-demand never late" vs errors (on-demand jobs can still be `failing`). The extension of the failure window for jobs with cadence longer than 24 hours is an author addition, flagged in Section B (R3).

## Unverified dependencies

- Read-only access paths to the automation hub's store, system timers and journal, the assistant's job file and execution history copy, and optionally database cron history.
- Registry table and dispatcher with collector-only `report` and human-only `set`.
- Scheduler for the 5-minute collector; owner notification channel.
- External heartbeat for the collector itself.

## Risk concerns

- Broad read access to operational data: mitigated by read-only handles or private copies, bounded window queries, redacted and truncated error excerpts, and no execution payloads.
- False greens: mitigated by R1 (staleness at read time), R2 (separate success timestamp), R6 (no inference for unmeasured jobs) and the read path refusing to return an empty "all good" on error. The external heartbeat alerts if the collector itself stops.
- False alarms: mitigated by R3, R4, R5, R7, R8 and the S6 sanity challenge.
- Alert fatigue: transition-only alerts, deduplication and summaries above 10 per cycle.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The core flow runs in the reference implementation; not every control described here has evidence yet.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour (source variants are complementary, not fallbacks).
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi and host-level read-only access**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O: R1 cannot be weakened; no write access to scheduler stores).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
