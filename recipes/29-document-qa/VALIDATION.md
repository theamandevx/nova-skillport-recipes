# Validation report — Document Q&A (`nova.knowledge.document-qa` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Maximum document size, per-question token cap and tier cost-approval rule (Section E).
- Schema registry IDs: every `schema://` reference in the manifest is defined in Section D as a compact JSON Schema, but none is registered with Skill Port yet.
- `identity_provider` for every tool in the manifest (`TBD`).

## Contradictions

None found. Checked: coverage gate (no LLM answer below threshold) vs `not_covered` counted as `completed`; "no automatic tier switch" vs the retry policy (retries stay on the same tier); compensation delete (own pending document) vs "delete requires approval" (approval applies to user documents, not the run's own unfinished write).

## Unverified dependencies

- PostgreSQL with pgvector and an HNSW cosine index in the target runtime.
- A 384-dimension embedding function with the same model as any existing index.
- Answer model tiers (cheap default; medium and pro optional) and their data-handling terms.
- The unit of the chunk window (tokens vs characters) in the reference implementation — to confirm (Section Q).

## Risk concerns

- Confidential text leaves the database for embedding and answering; operators with regulated data must pick providers accordingly.
- Hallucinated answers: mitigated by the coverage gate, per-sentence citation checks and the optional independent LLM check that can only downgrade.
- Threshold miscalibration could hide real answers (too high) or admit weak ones (too low); calibration per embedding model is an open item.
- Prompt injection through ingested documents: passages are treated as data; a test is listed in Section P.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. Only the happy path (known-answer retrieval at 0.914) has been observed, and only in the reference implementation.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour (no tier or embedding fallback).
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
