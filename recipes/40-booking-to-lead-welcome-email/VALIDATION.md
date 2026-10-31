# Validation report — Booking to lead, with a welcome email (`nova.sales.booking-to-lead-welcome-email` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Approval object format and how Skill Port binds an approval to a content hash (Sections G, Q).
- Default business hours and daily cap per owner; sales-data retention (Sections E, Q).
- `identity_provider` for every tool in the manifest (`TBD`).

## Contradictions

None found. Checked: autonomous send vs the package rule that third-party contact needs approval (resolved by a narrow, hash-bound, once-per-campaign-version pre-approved scope, as the conventions allow); stage monotonicity vs the cancellation rule (the only backwards move is explicitly listed); at-least-once events vs exactly-one email (idempotency on event, lead, enrolment and send key).

## Unverified dependencies

- Booking system outbox emitting created and cancelled events (the reference implementation used a database trigger; the outbox form is a hardening).
- Transactional email provider with idempotency keys and a verified sending domain.
- Inbound reply, bounce and opt-out handler that pauses or suppresses enrolments.
- Sales store functions for leads, campaigns, approvals, sequences, touches and suppression.
- Owner notification channel.

## Risk concerns

- Third-party email without per-message approval. Mitigated by the pre-approved scope, template hash binding, no LLM content, suppression checks at enrolment and send, business hours, daily cap and one send per booking.
- Pipeline pollution by internal or test bookings. Mitigated by the guard and the rule that an unavailable guard blocks rather than assumes external.
- Testing against live calendars can email real people; Section P forbids it.
- Lawful basis for the welcome varies by jurisdiction; the owner confirms it at approval.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The happy path has only been observed in the reference implementation.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour.
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi** (including the pre-approved scope).
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
