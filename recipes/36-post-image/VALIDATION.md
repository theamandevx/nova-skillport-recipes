# Validation report — Post image (`nova.content.post-image` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Per-image cost cap, daily image budget and a date-stamped provider price table (Section E).
- Retention period for rejected images (Sections O, Q).
- `identity_provider` for every tool in the manifest (`TBD`).
- Whether a timed-out generation is billed, per provider (Section F).

## Contradictions

None found. Checked: "no silent fallback" vs the provider-not-ready route (stops `blocked`, offers the other variant only as a new explicit choice); A2 autonomy vs paid calls (bounded by the per-image cap, 3-per-post limit and approval above cap); "never delete" vs compensation (re-attach from history, no deletion).

## Unverified dependencies

- Image provider A (multimodal model with image output) and image provider B (fixed-size image API) through registered adapters.
- Content store with post drafts and an `attach_image` operation that keeps image history.
- Object storage bucket, usage ledger and owner notification channel bindings.
- Vision-capable verifier for rendered text and logo-like marks.

## Risk concerns

- Trademark and likeness: mitigated by the "no third-party logos, no real-person likeness" hard constraint and the S5 rule check; residual risk because generators can still draw logo-like shapes — the independent verifier and the user's review cover it.
- Misinformation: mitigated by the fact-support rule (every number and name must appear in the post).
- Spend: mitigated by explicit provider choice, per-image cap, daily budget and the 3-generations-per-post limit; prices must be loaded from the operator's date-stamped table.
- Crop loss: provider B has no native 4:5 or 16:9 sizes; mitigated by preset safe zones; residual risk if platforms change crop behaviour.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The core flow runs in the reference implementation; not every control described here has evidence yet.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour (provider A and provider B have `fallback_variant_id: null` by design).
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
