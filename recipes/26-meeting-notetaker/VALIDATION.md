# Validation report — Meeting notetaker (`nova.calendar.meeting-notetaker` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: catalogue status is *Built, blocked* (a dedicated bot account must be provisioned by the deployer), joining a meeting is unverified for this package, no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform. Only the downstream stages (transcription with chunking, diarization, minutes, private storage, recap) were observed in Nova's reference implementation.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Dedicated bot account per platform and the deployer's terms review (Section C prerequisites; blocks activation).
- Meeting-link host allowlist, consent policy per jurisdiction, recording retention period, per-run cost cap and per-tenant bot concurrency (Sections E, Q).
- Join adapters for platforms other than the reference one (Section Q).
- `identity_provider` for every tool in the manifest (`TBD`).

## Contradictions

None found. Checked: "Leave on demand keeps what was captured" vs "consent withdrawn discards everything" (resolved by the `leave` reason: `user_leave` keeps, `consent_withdrawn` discards); "local file deleted only after a confirmed upload" vs the discard path (discard deletes it explicitly in S7); hosted → local transcription fallback vs no silent increase of external transfer (local → hosted is never automatic); `blocked` for consent withdrawal vs `partially_completed` (withdrawal never keeps notes, so it cannot be partial).

## Unverified dependencies

- Dedicated bot account, signed in once by hand (automated sign-in is blocked by the platform).
- Headless browser runtime, virtual audio sink and encoder; meeting web UI selectors for join and chat announcement.
- Hosted or local Whisper-class speech-to-text; local diarization model.
- Private recording store with auth-gated playback; atomic schedule queue with a full unique index on calendar event ID.
- Calendar booking list (MCP), approval gateway, owner notification channel.
- Minutes capabilities from `nova.calendar.meeting-minutes` (themselves partially verified).

## Risk concerns

- Recording people: legal exposure in all-party-consent jurisdictions, some with statutory damages per participant. Mitigated by visible naming, a default-on chat announcement, no capture before admission and announcement, the discard path and per-meeting approval; the runtime must still enforce local law and the deployer should get legal review.
- Platform terms: automated or business-function accounts may conflict with a platform's terms; the deployer must review them before activation.
- Platform friction: unrecognised participants may be held for manual admission, and enterprise tenants may label or block third-party bots — runs will end `blocked` more often than other recipes.
- Silent failures: a failed virtual audio sink can record silence; mitigated by the 60-second and periodic audio-health checks.
- Biometrics: diarization is ephemeral; voiceprints and cross-meeting speaker recognition are prohibited.
- Data at rest: recordings are sensitive; private storage and a short retention period must be configured before activation.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. Joining, admission, announcement and leave behaviour have no evidence for this package; downstream transcription and minutes were observed only in the reference implementation. All tests must use sandbox meetings hosted by the tester with consenting test participants.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies); the join binding also depends on a deployer-provisioned bot account.
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour.
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`; evidence status `Unverified`).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O; consent and disclosure never change silently).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
