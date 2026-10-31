# Validation report — Deal revival nudges (`nova.sales.deal-revival-nudges` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Per-run token cap (Section E).
- `identity_provider` for every tool in the manifest (`TBD`).

## Contradictions

None found. Checked: "never contacts anyone" vs the manifest (no outbound-to-lead binding exists); send-then-flag vs unknown delivery (no flag, so the lead may repeat — disclosed); cooldown vs new activity (new activity makes a lead eligible again).

## Unverified dependencies

- Sales store functions for stalled leads, touches and flags.
- Complete touch logging across the owner's channels (assumed).
- Structured-output LLM (optional; template fallback).
- Owner notification adapter with idempotency keys.

## Risk concerns

- Misleading openers (wrong last interaction, invented urgency). Mitigated by grounding rules, an independent rule check of dates and subjects, and the template fallback.
- False "quiet" signals when the owner does not log calls or meetings; disclosed in Sections K and Q.
- Low overall risk: read-only pipeline access except the flag, owner-only messaging.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The happy path has only been observed in the reference implementation.

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
