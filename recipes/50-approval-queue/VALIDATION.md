# Validation report — Approval queue (`nova.ops.approval-queue` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present and every schema referenced in the manifest is defined inline in Section D. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- `identity_provider` for every tool in the manifest (`TBD`) — especially important here, because the "human owner principal" check depends on it.
- Audit retention period (Section E, `TBD`).
- Whether Skill Port / Vedaxi already offers an approval primitive that should be reused (Section Q).

## Contradictions

None found. Checked: "approval is single-use" vs revocation (revocation applies only while `approved`, before consume); chat fallback vs high-risk rule (fallback goes only toward the stricter `dashboard` variant); "safety does not depend on the sweep" vs expiry (consume and decide check deadlines themselves); `consume` "no retry" vs idempotency (an ambiguous consume is resolved by reading status).

## Unverified dependencies

- Transactional database with role-scoped grants that keep `approvals:decide` away from every automation identity.
- Owner dashboard with sign-in, CSRF protection and a side-effect-free GET confirmation page.
- Chat platform callback verification (chat-buttons variant).
- The activity log recipe (`nova.ops.activity-log`) for same-transaction audit entries.
- Scheduler for the expiry sweep.

## Risk concerns

- This is a security control: a flaw lets automations reach third parties without consent. Mitigated by human-only decide grants, scope-hash recheck at consume, single use, expiry, hashed single-use tokens, GET-confirm/POST-decide, session requirement for high risk, and audit in the same transaction.
- A caller could write a misleading summary; the scope hash binds parameters, not wording, and the summary/preview mismatch rule is only a heuristic (Section Q).
- Link forwarding: a forwarded link plus the confirmation page could let another person decide low/medium approvals; mitigated by single use, short expiry, rate limits and the token-lock escalation; high risk requires the owner session.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. Only the core flow has been observed in the reference implementation; the remaining controls still need first evidence.

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
