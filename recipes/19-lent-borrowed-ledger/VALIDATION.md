# Validation report — Lent and borrowed ledger (`nova.finance.lent-borrowed-ledger` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present, and every schema referenced in the manifest is defined in Section D. The recipe is **ready for review, not for execution**. No capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform. Catalogue status is Built: chat-based logging with confirmation and an audit table ran in Nova's reference implementation. That is historical observation, not a current test run, and not every control described here has evidence yet (Section P lists the tests still needed).

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges (Section B; marked `UNVERIFIED` / `TBD`).
- Confirmation UI contract on Skill Port / Vedaxi (Sections G, Q).
- Retention policy for closed ledgers (Section E).
- `identity_provider` for every tool in the manifest (`TBD`).

## Contradictions

None found. Checked: "never deletes history" against corrections (void and replace, both audited); "confirm before logging" against the structured `form` variant (confirmation is still required); per-currency balances against summary replies (never netted).

## Unverified dependencies

- A conversation channel that can render a confirmation and return the user's choice.
- Ledger store with append-only entries and an audit table behind dispatcher functions.
- Operator-approved LLM provider (LLM parse variant only).

## Risk concerns

- A wrong direction or person corrupts a relationship balance. This is mitigated by mandatory confirmation that shows the balance before and after, clarification questions, and void-and-replace with audit.
- Double logging between the ledger and the finance hub. This is mitigated by the `not_ledger` route; legitimate overlap (card spend in the hub plus the receivable in the ledger) is explained in Section K.
- Privacy: counterparty names and amounts are personal data. They are tenant-scoped, and logs hold IDs only.

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
