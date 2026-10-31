# Validation report — Net worth overview (`nova.finance.net-worth-overview` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present, and every schema referenced in the manifest is defined in Section D (reused exchange-rate schemas are acknowledged there). The recipe is **ready for review, not for execution**. No capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform. Catalogue status is Built: the core flow (live balances, kind grouping, conversion to one currency) runs in Nova's reference implementation. Not every control described here has evidence yet (Section P lists the tests still needed).

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- External app provider, API version and rate limit (Sections E, Q).
- Registered binding IDs, schema registry IDs and version ranges (Section B; marked `UNVERIFIED` / `TBD`).
- `identity_provider` for internal tools (`TBD`).

## Contradictions

None found. Checked: read-only scope against the cached fallback (both sources are read-only, and cached values are always labelled); "one reporting currency" against FX honesty (no converted total from expired or unverified rates; per-currency subtotals are always shown); the `blocked` outcome against partial output (per-currency output is still returned and labelled).

## Unverified dependencies

- The external personal-finance app's REST API (list accounts, sync signal) and its authorization.
- Finance-hub account metadata and cached-balance function.
- FX snapshots from `nova.finance.daily-exchange-rates`.

## Risk concerns

- A misleading single number. Mitigated by showing flags before the total, labelling cached and stale inputs, and nulling totals on FX or verification problems.
- Exposure of a complete financial picture. Output goes to the owner only, the token stays server-side, and nothing is logged beyond counts.
- Sign conventions differ by provider. Handled by checking kind and sign and by a regression test.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The core flow has run in the reference implementation; not every control described here has evidence yet.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour (live → labelled cached, both read-only).
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
