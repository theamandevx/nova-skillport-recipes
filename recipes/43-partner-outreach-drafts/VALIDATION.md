# Validation report — Partner outreach drafts (`nova.sales.partner-outreach-drafts` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**. It is a **target design for rebuilding a removed module**: the reference implementation status is "Removed (rebuild planned)" and the evidence status is Unverified. No capability is registered with Skill Port, no binding is confirmed on Vedaxi, the rebuilt design has never run, and no Section P test has evidence.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Proof that the platform can restrict a mail grant to draft operations (Sections B, E, Q).
- Similarity threshold calibration for the rebuild's corpus; reply cap and follow-up spacing defaults; token cap; retention (Sections E, Q).
- Operator's opt-in jurisdiction list and consent-recording mechanism (Section Q).
- `identity_provider` for every tool in the manifest (`TBD`).

## Contradictions

None found. Checked: daily cap vs reply drafts (separate reply cap; follow-ups and first touches share the daily cap with follow-ups first); "never reactivate suppressed contacts" vs owner unsuppress (owner-only, lands on `paused`); follow-up timing vs drafts never sent (no observed SENT means no follow-up); dry run vs suppression on opt-out (dry run writes nothing, including suppressions — opt-outs are applied on the next real run).

## Unverified dependencies

- Draft-only mail adapter (create, get, list; no send).
- Thread observation able to match sent messages whose IDs differ from draft IDs.
- Contact store with review gate, warnings, inbox groups and suppression.
- Structured-output LLM for composition and reply classification.
- Approved offer configuration with an approval hash and an approved attachment.
- Owner notification channel.

## Risk concerns

- Drafts to third parties can be sent with one click. Mitigated by the review gate, never-send adapter, figure and claim-word rules, uniqueness gate and the owner's send action for every email.
- Opt-out and anti-spam law. Mitigated by opt-out line and postal address in every draft, immediate suppression, no reactivation, opt-in jurisdiction exclusion; legal sufficiency is the owner's responsibility.
- Uniqueness threshold carried over from earlier data may not transfer; must be re-calibrated before activation.
- Because the design has never run in this form, every behaviour in Sections H–M needs sandbox evidence before review can conclude.

## Tests still needed

Every row of Section P. None has evidence for the rebuilt design.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies; module currently removed).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour.
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi** (including the draft-only restriction).
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`; no evidence exists yet for the rebuild).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
