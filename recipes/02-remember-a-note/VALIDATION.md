# Validation report — Remember a note (`nova.memory.remember-a-note` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no memory binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform. Saving to memory and routing money to a ledger have only been observed in the reference implementation.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Default character limit per memory target and the token budget (Sections E, Q; `TBD`).
- How long replaced text is kept for restore (Section F; 7 days proposed, `TBD`).
- `identity_provider` for every tool in the manifest (`TBD`).
- All 8 schemas referenced in the manifest are defined inline in RECIPE.md Section D as draft JSON Schemas with matching `$id`s; they still need registering and validating against Skill Port's schema registry.

## Contradictions

None found. Checked: the explicit request as approval for an add vs approval required for replace and remove; reroute reported as `blocked` vs it being correct behaviour (Section M explains this deliberately); the two targets not being fallbacks for each other vs the capacity rule.

## Unverified dependencies

- Per-user persistent memory store with usage reporting in the target runtime.
- LLM for classification and drafting.
- Owner notification channel.
- Destinations for rerouted content (ledger, records, reminders) are separate recipes and are not provided by this package.

## Risk concerns

- Persistence: a wrong or sensitive entry is loaded into every later conversation. Mitigated by the screen, the fidelity check, read-after-write and an easy "forget that" path.
- Privacy: the rule-plus-LLM screen will miss some restricted data. Mitigated by rules overriding the LLM and by refusing on doubt; categories need jurisdictional review.
- Authority injection: a note could try to grant the agent permissions. Mitigated by the `permission_change` category and the hard rule that memory is data, never authority.
- Lost updates: concurrent writes could overwrite each other. Mitigated by serialising writes per user and re-reading before every write.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The restricted-data, permission-change and tenant-isolation tests are the highest priority.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour (targets are not fallbacks for each other).
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O; screen rules cannot be loosened without review).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
