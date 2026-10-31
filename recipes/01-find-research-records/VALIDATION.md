# Validation report — Find research records (`nova.research.find-research-records` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform. The web path has only been observed in the reference implementation; the scholarly and PDF variants are proposed and not built.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Token budget per run, overall time limit confirmation and library retention period (Sections E, O).
- Rate limits and contact or key requirements for the scholarly indexes (Section E; `TBD`).
- `identity_provider` for every tool in the manifest (`TBD`).
- All 10 schemas referenced in the manifest are defined inline in RECIPE.md Section D as draft JSON Schemas with matching `$id`s; they still need registering and validating against Skill Port's schema registry.

## Contradictions

None in behaviour. One representation gap in the §2.6 manifest template, to resolve with the platform schema: the template has one `binding_ref` per tool, so `nova.research.search.query` lists the metasearch binding at tool level and the scholarly and keyed-web variants rely on the bindings named in Section B. Checked as consistent: the grounding rule (never from memory) vs the `failed` outcome when no source is read; `save` default (true) vs the rule that an unverified brief is never saved; the keyed-web fallback vs the "no silent fallback" rule (it needs disclosure, and the manifest says so).

## Unverified dependencies

- Metasearch engine with a JSON API in the target runtime.
- HTTP fetcher with a public-address guard (redirect re-check included).
- LLM for synthesis and a separate verifier pass.
- Research library store with per-user scope and read-after-write.
- Scholarly-record indexes (OpenAlex, Crossref) — proposed only.
- PDF text extraction — proposed only.

## Risk concerns

- Misinformation: an LLM can misread a source. Mitigated by the rule-based number/quote check plus an independent verifier that can only remove findings.
- Untrusted content: fetched pages may contain prompt-injection text. Mitigated by treating page text as data and by a dedicated test.
- Server-side request forgery: mitigated by the public-address guard on every request and redirect.
- Privacy: query text leaves the tenant. Mitigated by the S2 redaction check and the approval for confidential queries.
- Copyright: mitigated by the 300-character quote cap and by not storing page bodies.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The happy path has only been observed in the reference implementation. The prompt-injection, fetch-guard and grounding tests are the highest priority.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies; scholarly, keyed-web and PDF variants not built).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour.
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
