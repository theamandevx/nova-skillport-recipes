# Validation report — Prepare an SMS (`nova.comms.prepare-an-sms` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**, and it is only **partly built** in the reference implementation: drafting exists and inbound SMS capture exists, but there is **no outbound SMS sender, no opt-out register, no inbound-reply handler and no delivery-receipt handling**. Evidence status is therefore `Unverified`. Only the hand-off variant (the user sends from their own phone) can run until the provider path is built and tested. No capability is registered with Skill Port.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- The SMS provider, its country coverage and sender registrations (not chosen).
- Per-message cost cap, daily caps and text retention (Section E; `TBD`).
- Opt-out keyword list and handling deadline per jurisdiction (Section Q; `TBD`).
- `identity_provider` for every tool in the manifest (`TBD`).
- All 13 schemas referenced in the manifest are defined inline in RECIPE.md Section D as draft JSON Schemas with matching `$id`s; they still need registering and validating against Skill Port's schema registry.

## Contradictions

None found. Checked: hand-off reported as `completed` vs the rule never to claim delivery (state stays `handed_off_unconfirmed` and is reported as such); the manifest's single binding reference on `nova.comms.sms.send.message` (`sms-handoff`) vs the provider variant's own binding (`sms-provider`, named in the variant description) — the platform must confirm whether per-variant bindings are supported; `max_attempts: 1` for send vs "no automatic re-send" (consistent).

## Unverified dependencies

- SMS provider REST API with registered senders per destination — **not built**.
- Opt-out register and inbound-reply handler — **not built**.
- Delivery-receipt handling — **not built**.
- Contact directory, assistant memory and owner notification channel bindings.
- Runtime policy engine for allowlist, quiet hours, sender registration and caps.
- Phone-platform handling of the `sms:` link body parameter.

## Risk concerns

- Legal: SMS consent, opt-out, quiet-hour and sender-registration rules differ by country and change; the provider variant must not activate until counsel and the provider confirm them per destination.
- Irreversibility: a delivered SMS cannot be recalled. Mitigated by hash-bound per-message approval and no automatic re-send.
- False success: provider acceptance is not delivery. Mitigated by reporting delivery only from receipts.
- Privacy: SMS is unencrypted in transit on carrier networks. Mitigated by forbidding secrets, codes and full account numbers in content.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. All provider-variant tests are blocked until the provider path is built. Highest priority once built: opted-out recipient, quiet hours, duplicate/replay and false-success tests.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED; provider send, opt-out register and status are not built**.
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour (no silent switch between hand-off and provider).
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`; not-built items marked).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow; the provider variant must stay inactive until built and tested**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
