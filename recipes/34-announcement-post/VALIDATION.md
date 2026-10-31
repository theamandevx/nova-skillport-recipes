# Validation report — Announcement post (`nova.content.announcement-post` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Per-draft token cap and queue retention (Section E).
- Schema registry IDs: every `schema://` reference in the manifest is defined in Section D as a compact JSON Schema, but none is registered with Skill Port yet.
- `identity_provider` for every tool in the manifest (`TBD`).

## Contradictions

None found. Checked: the brief used as base text vs "facts only from the fact sheet" (phrasing may be kept, facts may not be added); approval before an embargo vs "not ready to publish before the embargo" (recorded as approved-with-embargo); "no external side effects" vs the model call (a data transfer to the provider, declared in Section E, not a third-party action).

## Unverified dependencies

- Cheap-tier (extraction) and pro-tier (drafting) model availability; optional medium variants.
- Voice profile and content queue bindings.
- A separate publish flow for approved announcements (out of scope here).

## Risk concerns

- Misleading public statements if invented claims slipped through; mitigated by the verbatim fact sheet, the claim-to-brief map, the superlative check and the news-first check.
- Contractual/reputational exposure from naming partners or customers; mitigated by per-name confirmation (S6).
- Embargo breaches; mitigated by embargo-aware readiness in the queue.
- Heuristic detection of "someone else's news" and regulated topics can miss cases; the user's review is the backstop.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. Announcement drafting has only been observed in the reference implementation. Rollback/compensation is marked not applicable with a reason.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour (no tier switch).
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
