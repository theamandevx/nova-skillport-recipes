# Validation report — Meeting minutes (`nova.calendar.meeting-minutes` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform. The core extraction (summary, decisions, owner-tagged action items) has run in Nova's reference implementation; due dates, evidence spans, confidence flags and the grounding verifier are new in this recipe and untested anywhere.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Per-run token cap and the single-pass context budget per model (Section E).
- Retention period for transcripts and minutes (Sections E, O, Q).
- `identity_provider` for every tool in the manifest (`TBD`).

## Contradictions

None found. Checked: A2 autonomy vs "no external sends" (consistent — only the owner is messaged); verifier-unavailable path (stored with `verification: not_run`) vs completion rules (cannot be `completed`, Section M); chunked fallback vs "never escape a verification failure" (fallback limited to context-length errors).

## Unverified dependencies

- A mid-tier LLM with JSON output and a no-retention configuration.
- Minutes store with create/get by idempotency key.
- Calendar booking list (MCP) for attendees and meeting time.
- Owner notification channel.
- Upstream transcript producers (`nova.calendar.meeting-notetaker`, `nova.calendar.desktop-meeting-recorder`, a call engine) for the consent field.

## Risk concerns

- Hallucinated items, owners or dates: mitigated by evidence spans, the deterministic quote matcher, the separate verifier and the removal rule; residual risk remains on semantic support.
- Prompt injection inside transcripts: the transcript is wrapped as untrusted data; a dedicated test is listed.
- Privacy: transcripts are confidential personal data sent to an LLM provider; provider no-retention and tenant retention must be set before activation.
- People judgements: prohibited and checked, but the check relies partly on an LLM.
- Owner accuracy depends on upstream speaker labels, which may be missing for long audio.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. Only the happy-path core (summary, decisions, owners) has been observed in the reference implementation; due dates, flags, verifier, chunking and injection tests have not been run anywhere.

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
