# Validation report — Subscription audit (`nova.finance.subscription-audit` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**. No capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform. Catalogue status is Built: a monthly digest ran in Nova's reference implementation. That is historical observation, not a current test run, and not every control described here has evidence yet (Section P lists the tests still needed).

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges (Section B; marked `UNVERIFIED` / `TBD`).
- Report retention period and LLM token cap (Section E).
- Schemas: none missing — every `schema://` reference in the manifest is defined as a JSON Schema with a matching `$id` in Section D; reused sibling-recipe schemas are acknowledged there.
- `identity_provider` for every tool in the manifest (`TBD`).

## Contradictions

None found. Checked: "read-only" against storing a report (the report is the deliverable; financial data is never written); LLM suggestions against computed numbers (number lock plus template fallback); converted totals against stale FX (expired FX → no converted total). Section L is marked not applicable because the recipe makes no consequential decision; the standard requires escalation for consequential actions only.

## Unverified dependencies

- Read access to hub records, the payments log and the obligation schedule.
- FX snapshots from `nova.finance.daily-exchange-rates`.
- Operator-approved LLM provider (optional).
- Owner channel.

## Risk concerns

- Misleading totals (accounts not connected, possible series). Mitigated by listing included accounts, excluding "possible" series from totals, and independent recomputation.
- Over-confident advice. Mitigated by the no-usage-claims rule and question-style suggestions.
- Privacy: merchant names and amounts go to the LLM; account numbers are never sent.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The core flow has run in the reference implementation; not every control described here has evidence yet.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour (llm → template, both read-only).
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
