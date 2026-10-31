# Validation report — Dismiss an attention item (`nova.inbox.dismiss-an-attention-item` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Owner-interface contract for confirmation and Undo (Section Q).
- Audit retention period (Sections E, Q).
- `identity_provider` for every tool in the manifest (`TBD`).

## Contradictions

None found. Checked: "single owner click needs no approval" vs "bulk needs confirmation" (threshold: assistant-built selection or more than 10 items); dismiss vs triage integrity (only two fields may change, verified by hash); undismiss vs the digest (undismissed items reappear only if still unnotified and inside the window).

## Unverified dependencies

- Attention view shared by the dashboard and the digest (recipe 09).
- Triage results (recipe 08).
- Activity log binding.
- Authenticated owner sessions in the owner interface.

## Risk concerns

- Hiding an important email by mistake. Mitigated by reversibility, Undo, confirmation for bulk or assistant-built selections, and audit entries.
- Selector over-matching from a chat request. Mitigated by showing matches before applying and the 50-item cap.
- Bulk confirmation and the audit entry are additions not observed in the reference implementation.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. Single dismiss and undismiss have only been observed in the reference implementation.

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
