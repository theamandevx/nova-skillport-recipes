# Validation report — Voiceover (`nova.content.voiceover` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Per-run character cap, cost cap and price table (Section E).
- Provider's billable-character rules and clone duration limits (Sections E, Q).
- WER thresholds per language (Section H, S11).
- Retention periods for audio and consent statements (Sections E, Q).
- `identity_provider` for every tool in the manifest (`TBD`).

## Contradictions

None found. Checked: A2 synthesis vs consent rules (A2 applies only to stock or already-consented voices; cloning is A0/A1); allowlist fallback vs "no silent substitution" (allowlist is a declared read-only variant, stock voices only, disclosed); "never delete" vs revocation handling (disable only; provider-side deletion is an owner action outside the recipe).

## Unverified dependencies

- Text-to-speech provider with voice catalogue, quota and history endpoints; key permissions determine which reads work.
- Consent store and the owner's verified consent flow (external to this recipe).
- Database function that writes the audio record and usage row in one transaction.
- Independent speech-to-text verifier.

## Risk concerns

- Impersonation and biometric data (cloning): mitigated by owner-granted consent covering cloning and purpose, identity verification, per-clone approval, the ban on public or scraped audio, and disabling on revocation. Residual: the recipe cannot itself prove the sample speaker is the consenting owner — the consent flow must.
- Spend: mitigated by quota check, character and cost caps, duplicate check and single-writer cost recording.
- Fidelity: mispronounced names or numbers; mitigated by the transcribe-back check.
- Legal: AI-speech labelling and cloning rules differ by jurisdiction; the runtime must enforce them.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The core flow runs in the reference implementation; not every control described here has evidence yet.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour.
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi and the external consent flow**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O: consent rules can never be relaxed by an automated revision).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
