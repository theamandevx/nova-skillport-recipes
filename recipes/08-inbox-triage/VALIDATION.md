# Validation report — Inbox triage (`nova.inbox.inbox-triage` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Per-run token cap (Section E) and the agreement threshold for the sampled challenge (Section K).
- Retention period for triage fields and run records (Sections E, O).
- `identity_provider` for every tool in the manifest (`TBD`).

## Contradictions

None found. Checked: "never store bodies" vs the optional `reply_suggestion` (derived text, not the body; its removal is an open question in Section Q); snippet-only fallback vs least privilege (it reduces access); marker separation vs exclusion of transaction-tagged mail (read-only use of the other marker).

## Unverified dependencies

- Mail provider API with metadata listing and single-message read under a read-only scope.
- Metadata-only store with RFC Message-ID deduplication and a dispatcher that enforces tenant scope.
- Structured-output LLM provider with acceptable data-processing terms.
- Money classifier tags and obligations store (read-only inputs to the exclusions).
- Owner notification channel and the scheduler with a single-flight lock.

## Risk concerns

- Privacy: mail content reaches an LLM provider. Mitigated by transient reads, no body storage, disabled execution-data saving and a read-only provider scope.
- Prompt injection from email content. Mitigated by untrusted-data wrapping and S5 output checks.
- Two refinements are not in the reference implementation (grace period, attempt limit with parking); they need their own evidence.
- Classification accuracy is unproven at scale; only spot checks exist in the reference implementation.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The happy path has only been observed in the reference implementation.

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
