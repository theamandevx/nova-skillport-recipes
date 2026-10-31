# Validation report — Log transactions from SMS (`nova.finance.log-transactions-from-sms` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**. No capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform. Catalogue status is Built: the server path ran in Nova's reference implementation with test messages. Not every control described here has evidence yet (Section P lists the tests still needed).

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Whether Skill Port / Vedaxi supports inbound webhooks and per-installation token issuance (Sections B, Q).
- Registered binding IDs, schema registry IDs and version ranges (Section B; marked `UNVERIFIED` / `TBD`).
- Daily LLM cap and ledger retention (Section E).
- Schemas: none missing — every `schema://` reference in the manifest is defined as a JSON Schema with a matching `$id` in Section D; reused sibling-recipe schemas are acknowledged there.
- `identity_provider` for most tools in the manifest (`TBD`).

## Contradictions

None found. Checked: "respond quickly" against "process synchronously" (a 25-second budget; `retryable` + 503 when it cannot finish); idempotency against forwarder retries (the `processing / retryable / done` states make retries safe without losing messages); the auto-log money rule (the same bounded standing approval as the email logger).

## Unverified dependencies

- SMS-forwarder app behaviour (fields, retries, custom headers) on the user's phone.
- Inbound HTTPS webhook with header authentication on the target platform.
- Operator-approved LLM provider.
- Finance hub router, review queue and external-app sync declared in `nova.finance.log-transactions-from-email`.

## Risk concerns

- Public endpoint: brute force or replay. This is mitigated by constant-time token checks, rate limits, burst blocking and idempotency keys.
- Privacy: personal SMS and one-time codes. These are filtered before the LLM, and no text is persisted.
- Spoofed bank sender IDs. A message is logged only with a supported amount, account and reference; uncertain ones go to review.
- Double counting with email alerts. This is mitigated by the same-reference and ±3-day manual checks; fee differences still go to review.

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
