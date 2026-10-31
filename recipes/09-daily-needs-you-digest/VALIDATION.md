# Validation report — Daily "needs you" digest (`nova.inbox.daily-needs-you-digest` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Retention period for digest records (Section E).
- Escalation path when the owner channel itself is failing (Sections L, Q).
- `identity_provider` for every tool in the manifest (`TBD`).

## Contradictions

None found. Checked: "nothing dropped" vs "nothing sent twice" (resolved by adapter idempotency; when the adapter cannot confirm, the recipe prefers a disclosed possible duplicate over a silent drop); overflow items vs marking (only listed items are marked); quiet day vs stale triage (a stale-triage note is sent instead of silence).

## Unverified dependencies

- Owner notification adapter that records idempotency key → message ID (assumed; the reference implementation sent without it).
- Digest record store (a hardening beyond the reference implementation).
- Triage results (recipe 08) and the dismiss flag (recipe 11).
- Scheduler running in the owner's time zone.

## Risk concerns

- Privacy: subjects and summaries reach the chat provider. `minimal` mode and private-channel verification reduce exposure.
- Duplicate or dropped digests on ambiguous delivery. Mitigated by the prepared record, key-based reconciliation and send-then-mark.
- A quiet digest can hide broken triage; S8 and Section K require a staleness check.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The happy path and quiet day have only been observed in the reference implementation.

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
