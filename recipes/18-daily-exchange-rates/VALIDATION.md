# Validation report — Daily exchange rates (`nova.finance.daily-exchange-rates` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present, and every schema referenced in the manifest is defined in Section D. The recipe is **ready for review, not for execution**. No capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform. Catalogue status is Built: the daily refresh and cross-check ran in Nova's reference implementation. That is historical observation, not a current test run, and not every control described here has evidence yet (Section P lists the tests still needed).

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Chosen rate providers, their terms (attribution, rate limits, commercial use) and endpoint bindings (Sections B, Q) — operator decision.
- Registered binding IDs, schema registry IDs and version ranges (Section B; marked `UNVERIFIED` / `TBD`).
- Snapshot retention beyond the 400-day minimum (Section E).
- `identity_provider` for internal tools (`TBD`).

## Contradictions

None found. Checked: the secondary-only fallback against "no silent fallback" (it is declared, read-only and always labelled `unverified`); A3 autonomy against policy (reference data only, no personal data, no money); append-only storage against correction needs (newer snapshots supersede older ones).

## Unverified dependencies

- Two independent rate sources reachable over HTTPS with the configured currencies.
- Egress policy limited to the two hosts.
- Snapshot store and owner channel.
- Consumers that honour the conversion contract (`nova.finance.subscription-audit`, `nova.finance.net-worth-overview`).

## Risk concerns

- Wrong or stale rates would mislead totals. This is mitigated by cross-checks, the status model and the rule that consumers never show a plain converted total from expired or unverified rates.
- "Independent" sources that share an upstream feed. Section K requires the operator to check this, and S1 blocks identical configurations.
- Mid-market rates mistaken for bank rates. Consumers must label rates informational.

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
