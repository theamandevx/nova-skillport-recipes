# Validation report — Book a meeting (`nova.calendar.book-a-meeting` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Operator cap for optional LLM agenda drafting (Section E).
- `identity_provider` for every tool in the manifest (`TBD`).
- Schema registry IDs: every schema the manifest references is defined inline in Section D, but none is registered.

## Contradictions

None found. Checked: owner's ability to book at short notice vs the past-time guard (a 5-minute hard floor applies to everyone); compensation cancel vs "cancel needs approval" (allowed here only for bookings with no external event, so nobody is notified, and only at the user's choice); sandbox autonomy A2 vs the approval rule (sandbox cannot notify anyone because `send-none` has no fallback to `send-all`).

## Unverified dependencies

- Calendar database with an atomic booking function and idempotency keys.
- External calendar provider with event write, attendee notifications, a no-notifications mode and a private property for the booking key.
- Calendar-native video links and/or a self-hosted video-room provider.
- Contact directory, approval gateway and owner notification channel bindings.

## Risk concerns

- High-risk external side effect: guests receive invitation emails from the user's calendar. Mitigated by the per-booking approval bound to a content hash, directory-only guest emails, the guest cap and one-event-per-booking idempotency.
- Testing hazard: any test against a real calendar emails the organizer and guests, and a persisted test booking can be picked up by background jobs. Mitigated by mandatory `sandbox` mode, fictional guest domains, and rolled-back transactions.
- Wrong guest from speech recognition. Mitigated by fuzzy-match confirmation and the ban on inferred emails.
- Time-zone mistakes. Mitigated by explicit conversion, round-trip checks and per-party renderings in the approval and confirmation.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The happy path has only been observed in the reference implementation.

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
