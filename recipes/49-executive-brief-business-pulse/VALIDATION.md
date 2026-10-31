# Validation report — Executive brief and business pulse (`nova.ops.executive-brief-business-pulse` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present and every schema referenced in the manifest is defined inline in Section D. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- `identity_provider` for every tool in the manifest (`TBD`).
- LLM cost cap for the headline variant and run-record retention (Section E, `TBD`).
- Source adapter contracts (meetings, triaged mail, payment schedule, pipeline, activity, snapshots) are declared as dependencies; their own recipes or registry entries must supply them.

## Contradictions

None found. Checked: "quiet if nothing material" vs honest status (a skip with an unavailable source is `partially_completed`, Section M); "numbers never from an LLM" vs the `llm-headline` variant (the headline may only repeat numbers present in the facts and falls back to the deterministic headline); "one brief per period" vs re-runs after delivery failure (only `failed` claims can be re-claimed).

## Unverified dependencies

- Read-only source adapters for calendar, triaged mail, payment schedule, pipeline, activity log and external-project snapshots.
- A scheduler that fires in the owner's local time across daylight-saving changes.
- Owner channel with delivery receipts; optional LLM provider with recorded transfer consent.

## Risk concerns

- Concentration of personal and financial summaries in one message: mitigated by owner-only delivery, metadata-only mail, masked account numbers and no message text in run records.
- Misleading silence: mitigated by per-source freshness labels, the "source unavailable for 2 periods" note, and the zero-everywhere "check source" rule.
- LLM hallucinated figures: mitigated by the deterministic body, number-to-fact verification and the fallback.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The daily brief and the weekly pulse have only been observed in the reference implementation; the provenance labels, freshness SLAs and LLM-headline verification are prescribed designs that still need first evidence.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour.
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
