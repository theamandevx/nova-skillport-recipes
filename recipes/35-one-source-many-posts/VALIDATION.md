# Validation report — One source, many posts (`nova.content.one-source-many-posts` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Schema registry IDs: every `schema://` reference in the manifest is defined in Section D as a compact JSON Schema, but none is registered with Skill Port yet.
- Per-run token cap and queue retention (Section E).
- `identity_provider` for every tool in the manifest (`TBD`).

## Contradictions

None found. Checked: one set vs per-draft approval (approving one draft never approves its siblings; edits void only that draft's approval); attribution vs X's no-link rule (credit is given as text, the link is omitted); skipped drafts vs `completed` (any skipped draft makes the run `partially_completed`).

## Unverified dependencies

- Cheap-tier (planning) and pro-tier (drafting) model availability; optional medium variants.
- HTTP fetcher behaviour on script-rendered or paywalled pages (the user is asked for pasted text).
- Voice profile and content queue bindings.
- Multi-draft sets in one run — limited evidence in the reference implementation (Section Q).

## Risk concerns

- Error multiplication: one misread source yields several wrong drafts; mitigated by verbatim key-point spans and per-draft grounding checks.
- Copyright and attribution for third-party sources; mitigated by mandatory attribution, a 25-word copy limit and a licence escalation.
- Distortion in short form (X); mitigated by the caveat challenge in Section K and per-draft review.
- Platform cost and reach on X: links excluded by the X profile.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. Per-platform drafting has only been observed in the reference implementation; set generation needs recorded runs. Rollback/compensation is marked not applicable with a reason.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour (no tier switch; only LinkedIn and X profiles declared).
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
