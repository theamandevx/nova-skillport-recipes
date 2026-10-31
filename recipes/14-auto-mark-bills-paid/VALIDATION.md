# Validation report — Auto-mark bills paid (`nova.finance.auto-mark-bills-paid` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**. No capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform. Catalogue status is Built: the matching rules ran in Nova's reference implementation, including rolled-back tests. That is historical observation, not a current test run, and not every control described here has evidence yet (Section P lists the tests still needed).

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Retention period for the payments log (Section E).
- Schemas: none missing — every `schema://` reference in the manifest is defined as a JSON Schema with a matching `$id` in Section D; reused sibling-recipe schemas are acknowledged there.
- Failed-payment classification in the upstream extractor, which the presumed-paid sweep depends on (Section Q).
- `identity_provider` for every tool in the manifest (`TBD`).

## Contradictions

None found. Checked: "no balance changes" against the commit scope (no finance-hub scope is requested); presumed-paid against honest status (it is always labelled presumed, and it is reversible); fallback between match variants (all read-only, so no read→mutate switch).

## Unverified dependencies

- Payment-confirmation hand-off from `nova.finance.upcoming-payments-radar` (the radar's extractor).
- Obligation store with history (for restore) and a cycle guard.
- Payments-log store with a unique `source_ref`.
- Owner channel (optional).

## Risk concerns

- A wrong settlement hides a real unpaid bill. This is mitigated by reference-first matching, the four-condition fallback with a uniqueness rule, the timing gate, the user link for ambiguity and easy reopening.
- Presumed-paid can be wrong when an auto-charge fails. It is labelled presumed and reopened on a failed-payment notice; detection depends on upstream classification.
- Incoming revenue notices mistaken for payments. These are excluded by `direction = receive`.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The core flow has run in the reference implementation; not every control described here has evidence yet.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour (read-only match chain).
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
