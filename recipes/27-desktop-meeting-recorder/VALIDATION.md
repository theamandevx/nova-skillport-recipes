# Validation report — Desktop meeting recorder (`nova.calendar.desktop-meeting-recorder` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform. Catalogue status is "Built, blocked": capture has not been tested on the target computer and the sleep-safe stop waits for tests T1–T6 there.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- How Skill Port represents a local-IPC operation (`protocol: other` in the manifest) — no approved protocol value is known.
- Maximum recording length (Section D, E) — an open product decision.
- Per-recording cost cap, free-disk threshold, extraction token limit and the retention period for transcripts and minutes (Sections B, E, O).
- `identity_provider` for every server-side tool in the manifest (`TBD`).

## Contradictions

None found. Checked: two-track default vs no silent fallback to mic-only (fallback is user-chosen only); `ended_by_sleep` vs completion status (always `partially_completed`); `503` handling vs unpairing (never unpairs); "audio never stored server-side" vs the transient relay (relay discards audio after the response); local spool deletion vs "Delete not allowed" (only local spool is deletable; server-side records are not).

## Unverified dependencies

- Desktop OS app-audio capture API and signed-build permission prompt (reference: macOS 14.4+ process taps) — built, not tested on the target computer.
- Sleep-safe stop logic — written and reviewed, not applied.
- Hosted Whisper-class STT provider behind the relay; LLM for extraction and grounding.
- Pairing, device-token allowlist, minutes ingest and minutes store bindings (observed in the reference only).
- Owner notification channel.

## Risk concerns

- Recording people is high risk. Mitigated by the no-silent-capture invariant (press-only start, visible menu-bar and OS indicators, no auto-resume, stale-press discard), the consent reminder defaulting on, and transient-only server handling of audio.
- Legal: some jurisdictions require every participant's consent; the user remains responsible and the reminder cannot be disabled where policy requires it.
- Privacy: local audio exists for up to the spool retention (default 7 days); transcripts and minutes are personal data; retention must be set before activation. Diarization labels must never become voiceprints.
- Accuracy: "Others" merges all remote speakers in group calls; grounding check mitigates invented items but cannot recover gaps.
- Power: an app-level recording cap does not bound the OS audio daemon's own power assertion.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi, and capture plus T1–T6 need evidence on the target computer. Only the server-side `403 forbidden_for_device` and `503 auth_unavailable` behaviours and transcript ingest have been observed in the reference implementation.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies; local-IPC protocol value unconfirmed).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour (no silent fallbacks; ineligible variants marked).
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi and the OS permission system**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`; capture untested on the target computer).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls, including the press-only start and indicators (Section O).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
