# Validation report — Competitor news monitor (`nova.knowledge.competitor-news-monitor` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Skill Port's scheduler interface for the recurring job.
- Per-run token cap, per-site politeness delay and report retention (Section E).
- Schema registry IDs: every `schema://` reference in the manifest is defined in Section D as a compact JSON Schema, but none is registered with Skill Port yet.
- `identity_provider` for every tool in the manifest (`TBD`).

## Contradictions

None found. Checked: optional `medium-alt` fallback vs "no silent fallback" (it is declared, approved at installation, policy-equivalent and disclosed; without it a provider failure is `failed`); `no_new_coverage` counted as `completed` vs "never pad" (honest empty result); excerpts stored vs "full text transient".

## Unverified dependencies

- Self-hosted SearXNG and its upstream engines; adequacy of news coverage for the user's market.
- Fetcher behaviour on script-rendered pages (dropped as too little text).
- Mid-tier model availability and, if declared, a second provider for `medium-alt`.
- Research-report store, owner channel and job-run registry bindings.
- Scheduled per-competitor runs — limited evidence in the reference implementation (Section Q).

## Risk concerns

- Reputational/legal: summaries about real companies; mitigated by verbatim-quote and specifics grounding, attribution, no opinions, and the "needs your review" flag for allegations and litigation.
- Hallucination: mitigated by the provenance check (cited URL must be fetched in this run) and deterministic S7 rules.
- Copyright: only ≤ 300-character excerpts and links are stored.
- Prompt injection through fetched pages: pages treated as data; test listed.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The pipeline has been observed only in the reference implementation; a recorded scheduled-run set is still needed. Compensation is marked not applicable with a reason.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour (`medium-alt` declared-only and disclosed).
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
