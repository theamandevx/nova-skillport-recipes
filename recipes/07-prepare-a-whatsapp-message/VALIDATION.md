# Validation report — Prepare a WhatsApp message (`nova.comms.prepare-a-whatsapp-message` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform. In the reference implementation the linked-device channel works and delivers messages after the user's yes, but third-party sends through it are unverified, and the Business Platform variant is not built.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Business Platform number, templates, pricing and messaging limits (Sections E, Q; `TBD`).
- Daily caps, quiet-hours defaults and text retention (Section E; `TBD`).
- `identity_provider` for every tool in the manifest (`TBD`).
- All 11 schemas referenced in the manifest are defined inline in RECIPE.md Section D as draft JSON Schemas with matching `$id`s; they still need registering and validating against Skill Port's schema registry.

## Contradictions

None in behaviour. Two representation gaps in the §2.6 manifest template, to resolve with the platform schema: (1) the template has one `binding_ref` per tool, so `nova.comms.whatsapp.send.message` lists the linked-device binding at tool level and names the business-api and hand-off bindings in their variant descriptions; (2) the template has no per-variant authorization, so the tool lists both `whatsapp:send:personal` and `whatsapp:send:business` although each variant needs only its own scope (as Section B states). Checked as consistent: hand-off reported as `completed` but never as delivered; the purpose rule enforced at S3 and S7; no silent variant switch.

## Unverified dependencies

- Linked-device bridge with session-state reporting and a per-user outbound allowlist in the target runtime.
- WhatsApp Business Platform (send, opt-in records, window check, templates, status webhooks) — **not built**.
- Contact directory, assistant memory and owner notification channel bindings.
- Receipt handling for the linked-device variant (depends on what the bridge reports).

## Risk concerns

- Terms of service: automated messaging from a personal account can get the number restricted. Mitigated by limiting linked-device to personal one-to-one sends with an allowlist and daily cap, and routing customer-facing messages to the Business Platform.
- Account takeover: a pairing QR code grants account access. Mitigated by never automating re-pairing and showing codes only to the owner over an authenticated channel; session credentials held as secrets.
- Irreversibility: messages may be read immediately. Mitigated by hash-bound per-message approval and no automatic re-send.
- Availability: the session can drop for hours. Mitigated by honest `blocked` status, re-pair instructions and hand-off.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. Business Platform tests are blocked until that variant is built. Highest priority: purpose rule (business never via linked-device), session-logged-out, allowlist and false-success tests.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED; Business Platform tools not built**.
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour (no fallback between variants without a new approval).
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi; per-variant scopes need platform support**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`; not-built items marked).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O; purpose rule and allowlist cannot be relaxed without review).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
