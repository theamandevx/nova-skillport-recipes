# Validation report — LinkedIn post in your voice (`nova.content.linkedin-post-in-your-voice` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform. Feature status is **Built, blocked**: drafting and the queue run in the reference implementation; publishing is blocked until the user's LinkedIn app and OAuth grant are approved and active.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- LinkedIn permission names and API version, confirmed against LinkedIn's current documentation (marked UNVERIFIED in Section B and the manifest).
- Daily publish cap, per-draft token cap and queue retention (Section E).
- Schema registry IDs: every `schema://` reference in the manifest is defined in Section D as a compact JSON Schema (cross-recipe reuse of the trending-topics-radar schemas is acknowledged in the text where it occurs), but none is registered with Skill Port yet.
- `identity_provider` for non-LinkedIn tools (`TBD`).

## Contradictions

None found. Checked: "Approve and publish now" in S7 is the per-post approval and is bound to the content hash (an edit voids it); scheduled posts produce reminders only and need a fresh publish approval; a blocked connection yields run status `blocked` while the draft stays queued (not reported as completed for `draft_and_publish`).

## Unverified dependencies

- The user's LinkedIn developer app with member-posting access, and an active OAuth grant — **currently the blocking dependency**.
- Long-lived (refreshable) LinkedIn access, which needs additional app approval from LinkedIn; without it, grants expire (about 60 days in the reference implementation).
- Pro-tier (and optional medium-tier) model availability.
- Content queue, voice profile, topic store, fetcher and owner reminder channel bindings.

## Risk concerns

- High-impact external side effect: public posts under the user's name. Mitigated by per-post, hash-bound approval, no automatic publish retry, no auto-publish of scheduled posts, and no alternative publish path.
- Authenticity: drafts must not invent the user's experiences, clients or results; enforced by the claims check.
- Accuracy: specifics must trace to the source or brief; launches must lead with what shipped.
- Platform terms: no engagement automation, outreach or browser automation.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. Drafting has only been observed in the reference implementation; the publish happy path cannot be tested until a LinkedIn test account with an approved app and active grant is available.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED**; the LinkedIn publish binding is additionally **blocked** on the user's app approval.
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour (no tier switch; no alternative publish path).
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi and on LinkedIn**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`; publishing blocked).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**; drafting can activate without publishing.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
