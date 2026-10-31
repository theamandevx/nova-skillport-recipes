# Validation report — External project watch (`nova.ops.external-project-watch` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present and every schema referenced in the manifest is defined inline in Section D. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- `identity_provider` for every tool in the manifest (`TBD`).
- Snapshot retention default needs operator confirmation (Section Q).
- The `watch-run@0.1.0` run record is defined in Section D but not attached to a capability output in the manifest (it is the recipe-level run envelope).

## Contradictions

None found. Checked: "no inbound credential" vs secret storage (Nova stores only the ingest secret it issued, never a credential for the external project); "alerts once per change" vs level-rule repeats (a repeat needs the value to at least double and is capped at once per 24 h); "202 after store" vs asynchronous alerts (the response confirms storage only, Section I).

## Unverified dependencies

- The user-installed sender (`external.rollup-sender`): read-only aggregate function, scheduler with HTTPS POST, and a secret store in the external project.
- An HTTPS webhook gateway able to verify HMAC signatures (or an adapter).
- Snapshot store with a unique constraint and append-only grants; a scheduler for the freshness check; owner channel; dashboard.

## Risk concerns

- Internet-facing endpoint: mitigated by HMAC with a timestamp window, size limit before parsing, strict schema, rate limit, and no detail in rejections.
- The `shared-header` variant is weaker (no signed timestamp); it is opt-in, confirmed explicitly and labelled on the dashboard.
- Sender-side objects live outside Nova's backups and inventory; the runbook must list them with disable and removal steps.
- Misleading green: mitigated by server-side age, stale detection, and the "suspicious all-zero" hold.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The happy path and baseline behaviour have only been observed in the reference implementation; the remaining rows still need first evidence.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour.
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
