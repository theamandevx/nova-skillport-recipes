# Validation report — Draft a reply (`nova.inbox.draft-a-reply` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Proof that the platform can restrict a mail grant to draft operations (Sections B, E, Q).
- Per-run token cap (Section E).
- `identity_provider` for every tool in the manifest (`TBD`).

## Contradictions

None found. Checked: "never sends" vs a provider scope that permits sending (resolved by requiring an adapter-level draft-only operation set, verified in S1); one-draft-per-email vs ambiguous create timeouts (reconciled with `find_draft`); holding reply vs the no-commitment rule (holding reply contains no date or promise).

## Unverified dependencies

- Mail provider draft endpoint with thread ID and custom threading headers.
- Adapter that exposes draft create, get and list but not send.
- Triage results and reply suggestions from recipe 08.
- Structured-output LLM and the independent verifier pass.
- Owner notification channel.

## Risk concerns

- A draft could be sent by the owner without reading it. Mitigated by the no-commitment rules, the independent verifier and short drafts.
- Malformed or spoofed sender addresses. Mitigated by the quote-aware parser, the well-formed rule at queue and precheck, and the same-domain `Reply-To` rule.
- Queue jam from a single failing item. Mitigated by per-item recording, backoff and parking; parking is a hardening not yet observed in the reference implementation.
- Personal mailboxes are excluded by a hard constraint.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The happy path has only been observed in the reference implementation.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour.
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi** (including the draft-only restriction).
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
