# Validation report — Find a saved memory (`nova.memory.find-a-saved-memory` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no memory or session-store binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform. Recall from memory and sessions has only been observed in the reference implementation.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Token budget (Section E; `TBD`) and session retention period (runtime setting; Section Q).
- `identity_provider` for every tool in the manifest (`TBD`).
- All 8 schemas referenced in the manifest are defined inline in RECIPE.md Section D as draft JSON Schemas with matching `$id`s; they still need registering and validating against Skill Port's schema registry.

## Contradictions

None found. Checked: an honest "not found" counted as `completed` vs the rule that a partial run is never reported as complete (not-found is only `completed` when every available store was searched; otherwise `partially_completed`); the recall recipe being read-only vs the offer to remember (handed to a separate recipe, not executed here).

## Unverified dependencies

- Per-user persistent memory store (read) in the target runtime.
- Per-user session store with full-text search and channel/time metadata.
- LLM for expansion, composition and an independent verifier pass.
- Owner notification channel with a private/shared flag supplied by the runtime.
- The dated entry prefix depends on the remember-a-note recipe; legacy entries are undated.

## Risk concerns

- Fabrication: an LLM may "remember" plausible details. Mitigated by the sentence → match mapping check, the verbatim-list fallback and the rule that dates appear only when recorded.
- Privacy: answers expose personal data. Mitigated by the private-channel rule and per-user binding.
- Untrusted snippets: past sessions can contain text other people wrote. Mitigated by treating snippets as data and labelling non-user roles.
- Retention artefacts: "not found" may reflect deleted history; the answer wording discloses this.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The not-found, undated-entry, tenant-isolation and false-success tests are the highest priority.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour (read-only fallback from memory to sessions only).
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
