# Validation report — Upcoming payments radar (`nova.finance.upcoming-payments-radar` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**. No capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform. Catalogue status is Built: the behaviour has run in Nova's reference implementation. That is historical observation, not a current test run, and not every control described here has evidence yet (Section P lists the tests still needed).

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Per-run LLM token cap (Section E) and retention period for verdicts and obligation history (Sections E, Q).
- Name and schema of the obligation change-history store used for compensation (Sections F, J).
- Schemas: none missing — every `schema://` reference in the manifest is defined as a JSON Schema with a matching `$id` in Section D; reused sibling-recipe schemas are acknowledged there.
- `identity_provider` for every tool in the manifest (`TBD`).

## Contradictions

None found. Checked: standing-scope autonomy (A2) against the money rules (records are reminders only; no balance or payment effect); "defer, never guess" against verdict marking (deferred items never get a verdict); exact-match cancellation dismissal against fuzzy matches (review only).

## Unverified dependencies

- Mailbox API with metadata listing and full-message reads, plus OAuth consent per mailbox.
- Operator-approved structured-output LLM provider.
- Obligation store with security-definer dispatcher functions and a unique `uniq_key`.
- Finance-hub account list with last-4 match hints.
- Runtime run lock and kill switch.

## Risk concerns

- Privacy: private email text goes to an LLM provider. This is mitigated by transient bodies, no persistence, operator approval of the provider and no runtime provider fallback.
- Missed payment: a wrong dismissal or a stale mailbox could hide a real bill. This is mitigated by exact-match-only auto-dismissal, per-mailbox freshness disclosure and needs-review defaults.
- Prompt injection inside emails: routing is server-enforced and email text cannot change scopes.
- Phishing: lures never create obligations, and no links are opened.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The core flow has run in the reference implementation; not every control described here has evidence yet.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour (the LLM alternate is operator-selected only).
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
