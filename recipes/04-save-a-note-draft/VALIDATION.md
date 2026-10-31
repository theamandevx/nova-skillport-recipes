# Validation report — Save a note draft (`nova.memory.save-a-note-draft` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, the notes-store binding is not confirmed on Vedaxi, and no Section P test has evidence on the target platform. Drafting and saving for confirmation have only been observed in the reference implementation.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Token budget, confirmation wait window and discarded-draft purge period (Sections E, Q; `TBD`).
- Whether the target store supports conditional updates on a content hash (Section Q).
- `identity_provider` for every tool in the manifest (`TBD`).
- All 8 schemas referenced in the manifest are defined inline in RECIPE.md Section D as draft JSON Schemas with matching `$id`s; they still need registering and validating against Skill Port's schema registry.

## Contradictions

None found. Checked: the package `app_id` (`nova.memory`) vs capability IDs under `nova.notes` (deliberate: notes are a separate store from persistent memory, packaged together); "draft saved" being the deliverable vs `completed` requiring a user response (an unanswered draft is `partially_completed`, stated in Section M); Discard as a status change vs "Delete: Not Allowed".

## Unverified dependencies

- Per-user notes store with a status field in the target runtime.
- LLM for structuring and an independent fidelity pass.
- Owner notification channel with reply options.

## Risk concerns

- Fidelity: paraphrase can shift meaning. Mitigated by the independent fidelity check, mechanical removal of added facts and mandatory user confirmation.
- Secrets in pasted text: mitigated by pattern rules that override the LLM and a redact-or-cancel choice; values are never logged.
- Wrong-version confirmation: mitigated by confirming only the content hash the user saw, with conditional updates.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The added-fact trap, secret redaction and confirm-after-change tests are the highest priority.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour (single variants; no fallback store).
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
