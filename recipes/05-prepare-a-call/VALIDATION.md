# Validation report — Prepare a call (`nova.comms.prepare-a-call` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Per-call cost cap default (Section E) and retention period for transcripts and recordings (Section E, O).
- `identity_provider` for every tool in the manifest (`TBD`).

## Contradictions

None found. Checked: approval scope (one call) vs no-redial rule; recording disclosure default (on) vs policy override; outcome mapping for not-connected calls (Section M).

## Unverified dependencies

- Telephony provider with bidirectional audio streaming and a permitted caller ID.
- Speech-to-text, text-to-speech and tool-calling LLM providers in the user's region.
- Contact directory, assistant memory and owner notification channel bindings.
- Runtime policy engine for allowlist, quiet hours and cost cap.

## Risk concerns

- High-risk external side effect: a real person is called. Mitigated by per-call approval, allowlist, quiet hours, one-call-per-approval idempotency and no automatic redial.
- Legal: recording disclosure and caller-ID rules vary by country; the runtime must enforce them.
- Privacy: transcripts and recordings are personal data; retention must be set before activation.
- Recap accuracy: mitigated by the independent transcript cross-check (Section K).

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
