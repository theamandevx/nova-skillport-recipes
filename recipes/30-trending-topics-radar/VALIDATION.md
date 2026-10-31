# Validation report — Trending topics radar (`nova.knowledge.trending-topics-radar` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Skill Port's scheduler interface for installing the recurring job (the recipe declares `schedule:create` at installation but cannot name the platform object).
- Per-source request rates, monthly note budget and topic retention period (Section E).
- Schema registry IDs: every `schema://` reference in the manifest is defined in Section D as a compact JSON Schema, but none is registered with Skill Port yet.
- `identity_provider` for every tool in the manifest (`TBD`).

## Contradictions

None found. Checked: both source variants running together vs "no fallback" (they are complementary and a failure is disclosed, never substituted); quiet-hour deferral vs digest idempotency (the deferred digest keeps the same key); the owner's pick updating status vs "Update allowed on the owner's action only".

## Unverified dependencies

- A public tech-news aggregator search API reachable from the target runtime, and its terms of use.
- A self-hosted SearXNG instance (optional) and the upstream engines it depends on.
- Topic store, owner notification channel and job-run registry bindings.
- A cheap-tier model for notes (optional).

## Risk concerns

- Low overall: public read-only sources, owner-only writes and messages.
- Misleading prominence: engagement-heavy ranking can favour sensational headlines; mitigated by relevance weighting and cross-source labelling (Section K).
- Note hallucination: mitigated by the title/snippet grounding rule, which nulls any unverifiable note.
- Silent failure of a scheduled job: mitigated by the run record on every path and the "stale is never healthy" rule.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The happy path has only been observed in the reference implementation. The compensation test is marked not applicable with a reason.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour (sources complementary; no note-tier switch).
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
