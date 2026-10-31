# Validation report — Meeting prep brief (`nova.calendar.meeting-prep-brief` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Schema registry IDs: every schema the manifest references is now defined inline in Section D (compact JSON Schema), but none is registered.
- Per-brief token cap and retention period for briefs and research records (Sections E, O).
- `identity_provider` for every tool in the manifest (`TBD`).
- Owner preference for re-briefing a rescheduled meeting (Section Q).

## Contradictions

None found. Checked: A2 autonomy with no approval vs "anything reaching a third party needs approval" (nothing reaches the attendee; delivery is owner-only); `ask-user` contact variant vs unattended scheduled runs (declared ineligible in scheduled runs); "three talking points" vs grounding rule (fewer are sent and disclosed when not all are supported); marker only after delivery vs once-per-booking (the message key also prevents a second send if the marker write fails).

## Unverified dependencies

- Prep-due query and once-per-booking marker on the calendar store.
- Contact directory, CRM, mail metadata (without bodies), minutes history and assistant memory bindings.
- Cited web-research capability (self-hosted metasearch, page fetch, LLM synthesis, research store).
- A mid-tier LLM for composition and a separate verification pass.
- Owner notification channel.

## Risk concerns

- Privacy: the brief aggregates personal data about a third party. Mitigated by owner-only delivery, no mail bodies, professional-context-only research, a sensitive-category filter and retention limits; the operator must confirm a lawful basis for attendee research per jurisdiction.
- Accuracy: same-name confusion and stale roles. Mitigated by email-first identity matching, organisation/domain tie-in for public claims, `identity_confidence`, mandatory citations and the independent S7 check.
- Prompt injection from fetched pages: mitigated by treating web content as quoted data and by the hard rule that the recipe can never contact the attendee.
- Testing harm: a persisted synthetic booking can be picked up by a polling workflow and email a real guest; tests are restricted to rolled-back transactions and fictional `@example.com` attendees in a sandbox tenant.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The own-data happy path has only been observed in the reference implementation (historical evidence); the public-section path has not been observed anywhere.

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
