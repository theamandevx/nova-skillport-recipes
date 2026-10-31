# Validation report — Cost per job (`nova.ops.cost-per-job` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`). The manifest uses a templated binding reference `binding://nova/usage/source-{variant}`; each variant's concrete binding is listed in its constraints.
- Replay-guard values, spike floor, ledger retention and price-table maintenance owner (Sections D, E, Q; illustrative defaults only).
- Expected source schemas per collector (field lists are deployment-specific).
- `identity_provider` for the collector service identity and readers (`TBD`).

## Contradictions

None found. Checked: "going forward only" vs rebaseline (rebaseline also moves forward, skipping a gap with approval — never backfills); "append-only" vs price corrections (adjustment rows, never edits); "honest zero" vs spike detection (billed spend drives alerts; zero-cost bases only produce an informational list-price spike); A3 autonomy vs guards (guards refuse; only a human can raise them or rebaseline).

## Unverified dependencies

- Read-only access to the hub's execution store and a private copy of the assistant's session store.
- Ledger and watermark in one database supporting a single transaction across both.
- Price table with per-tier billing confirmation; provider invoice exports for reconciliation (TBD).
- Scheduler and owner notification channel.

## Risk concerns

- Wrong numbers drive wrong decisions: mitigated by atomic commits, unique source references, S9 re-read, per-job-to-total reconciliation and labelled estimates.
- History replay after a lost watermark: mitigated by replay guards that refuse the whole batch.
- Silent schema changes in a source: mitigated by drift refusal before any read.
- Hidden spend on "free" tiers: mitigated by recording list price alongside the zero cost and requiring operator confirmation to flip.
- Content leakage: the ledger stores token counts only, never prompt or completion text.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The core flow runs in the reference implementation; not every control described here has evidence yet.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour (the two collectors are complementary, not fallbacks).
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi and host-level read-only access**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O: guards cannot be removed; the ledger cannot become editable).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
