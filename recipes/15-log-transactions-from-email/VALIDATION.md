# Validation report — Log transactions from email (`nova.finance.log-transactions-from-email` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**. No capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform. Catalogue status is Built: the behaviour has run in Nova's reference implementation. That is historical observation, not a current test run, and not every control described here has evidence yet (Section P lists the tests still needed).

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Format of the standing auto-log approval record, its default expiry and default per-currency caps (Sections D, Q).
- External app provider, API version, rate limit and field mapping (Sections E, Q).
- Schemas: none missing — every `schema://` reference in the manifest is defined as a JSON Schema with a matching `$id` in Section D; reused sibling-recipe schemas are acknowledged there.
- `identity_provider` for every tool in the manifest (`TBD`).

## Contradictions

None found. Checked the money rule "confirm before logging anything the user did not explicitly state" against auto-logging. They are reconciled by making auto-logging a narrowly bounded, user-granted standing approval (named accounts, per-transaction cap, high + high, new messages only). Everything else needs a per-item approval in the review queue. The `±3 days` duplicate rule is fixed, not configurable, as specified.

## Unverified dependencies

- Mailbox API (metadata + full read) with OAuth per mailbox.
- Operator-approved structured-output LLM provider.
- Finance hub: accounts with match hints, records with unique `(source_type, source_ref)`, review queue, router function.
- External personal-finance app REST API (create and find records) and its authorization.
- Approval store, run lock and kill switch.

## Risk concerns

- High: wrong records change displayed balances. This is mitigated by the strict gate, server-side recomputation, the S11 scope audit, the review default and user reversal.
- Double counting across manual entries, emails and SMS. This is mitigated by the ±3-day manual check and the same-reference check; fee/FX differences still need review.
- Double posting to an external API without idempotency keys. This is mitigated by the claim protocol and `verify_remote`.
- Privacy: email text goes to the LLM; record fields go to the external app. Bodies are never stored.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The core flow has run in the reference implementation; not every control described here has evidence yet.

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
