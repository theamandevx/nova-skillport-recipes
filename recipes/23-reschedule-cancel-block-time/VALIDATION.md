# Validation report — Reschedule, cancel or block time (`nova.calendar.reschedule-cancel-block-time` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Schema registry IDs: every schema the manifest references is now defined inline in Section D (compact JSON Schema), but none is registered.
- Per-run token cap, approval timeout default and retention period (Sections E, O).
- `identity_provider` for every tool in the manifest (`TBD`).
- How the runtime exposes propagation status to the agent (Section Q).

## Contradictions

None found. Checked: block needs no approval vs "anything reaching a third party needs approval" (blocks notify nobody); idempotent cancel of an already-cancelled booking reported `completed` vs "no change" (defined as `already_in_state`); `send-none` allowed in production only on the owner's explicit choice vs "never silently"; `busy_range` "keep" decision vs block creation (the meeting stays and the block is added around it, disclosed).

## Unverified dependencies

- Calendar store with slot engine, busy blocks and booking history rows.
- Single propagation point (database-queued outbound call) and an external calendar provider that emails attendees on update and delete.
- Approval gateway with one-tap token links; owner notification channel.
- Busy-sync status source.

## Risk concerns

- High-risk external side effect: guests are emailed on reschedule and cancel; emails cannot be recalled. Mitigated by per-change approval bound to the booking version, idempotency keys, no blind retries and no direct calls to the propagation operation.
- Concurrency with the guest self-serve page: mitigated by the booking-version check at apply time.
- Time-zone errors: mitigated by explicit IANA conversion, daylight-saving gap/fold detection and a deterministic formatter that owns every displayed time.
- Silent no-op blocks: mitigated by rejecting `range_start >= range_end`.
- Testing harm: a real calendar emails the organizer and guests on every create and cancel; tests are restricted to rolled-back transactions, database-only bookings, `send-none` and fictional `@example.com` guests.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The reschedule, cancel and block happy paths have only been observed in the reference implementation, and that evidence is historical.

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
