# Validation report — AI quota monitor (`nova.ops.ai-quota-monitor` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`). The manifest uses a templated binding reference `binding://nova/quota/{variant}`; each variant's concrete binding is listed in its constraints.
- Balance floors, runway day thresholds (7 and 2 are illustrative), quiet hours, repeat interval for unresolved critical alerts, snapshot retention (Sections B, E, H, Q).
- Per-provider read-only account scopes and whether usage reads consume billable quota (Sections E, Q).
- `identity_provider` per provider and for the collector service identity (`TBD`).

## Contradictions

None found. Checked: "observe only" vs the runtime account-usage variant (config-hash guard stops the variant on any change); "keep last good" vs "stale after 3 hours" (last good is retained but displayed as stale with its age, never as current); positive balance vs exhaustion errors (signal overrides, both shown); alert-once vs first observation (unlike job health, a first snapshot already above a threshold does alert, because the condition is real).

## Unverified dependencies

- Assistant runtime account-usage function (supported providers only) and provider usage or balance endpoints.
- Usage ledger from `nova.ops.cost-per-job` and operator-recorded purchases for the ledger-derived estimate.
- Error sources for exhaustion signals (the operator's own call logs or job records).
- Provider registry, snapshot store, scheduler and owner notification channel.

## Risk concerns

- Account-level read access: mitigated by read-only scopes and no purchase or routing permissions.
- Runtime side effects: a read through the runtime must not change its configuration; mitigated by the before/after configuration hash and stopping the variant on change.
- Misleading headroom: mitigated by stale ≠ empty (Q1), null-not-zero, last-good retention (Q2) and labelled estimates.
- Silent exhaustion: mitigated by balance runway levels and exhaustion signals; residual where a provider has neither a balance endpoint nor purchase records.
- False sense of redundancy: mitigated by the fallback-independence check (Q6).

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The core flow runs in the reference implementation; not every control described here has evidence yet.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour (`ledger-derived` is a declared, labelled read-only fallback for providers without a balance endpoint; window variants are chosen by configuration).
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi and each provider**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O: no purchase or routing permissions; Q1 cannot be weakened).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
