# Validation report — Activity log (`nova.ops.activity-log` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present and every schema referenced in the manifest is defined inline in Section D. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- `identity_provider` for every tool in the manifest (`TBD`).
- Retention period (Section E, `TBD`).
- The per-source meta-key allowlists are declared in `writer@0.1.0` but their contents are left to each writer's registration.

## Contradictions

None found. Checked: "append-only" vs retention (only the retention role deletes, only past the approved period, with a tombstone that keeps the chain verifiable); "never edit" vs corrections (corrections are new entries); "no fallback between variants" vs gateway outages (the writer retries the gateway with the same key).

## Unverified dependencies

- PostgreSQL-compatible database with triggers, security-definer functions and transaction-scoped locks.
- HTTPS logging gateway with per-writer credentials and source binding.
- Owner dashboard with sign-in for the feed.
- Scheduler for verification and retention; optional off-site anchoring via the backup recipe.

## Risk concerns

- The log concentrates personal activity: mitigated by owner-only feed access, masking of emails and phone numbers, and no bodies or payloads.
- Secrets leaking into logs: mitigated by redaction with a second scan and writer-level alerts on repeated redactions.
- Tampering by a database owner: mitigated by the hash chain and off-site anchoring of the last verified hash; without anchoring, a full rewrite by the database owner remains possible (Section Q).
- Same-transaction triggers can block domain writes if the logging function fails — a deliberate trade-off that needs strong tests.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. Triggers, the gateway and the searchable feed have been observed in the reference implementation; the remaining controls still need first evidence.

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
