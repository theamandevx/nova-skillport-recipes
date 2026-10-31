# Validation report — Lead sourcing (`nova.sales.lead-sourcing` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Per-run credit cap, LLM token cap, fit threshold and pending-lead retention (Sections E, Q).
- The operator's opt-in jurisdiction list (Sections C, Q).
- `identity_provider` for every tool in the manifest (`TBD`).

## Contradictions

None found. Checked: "verify before storing" vs quota exhaustion (unverified addresses are discarded, never stored); dedupe vs re-running an ICP (pending leads updated, reviewed leads skipped); "nothing contacted" vs `send_eligible_after_review` (a review hint only; contactability is set by the owner outside this recipe).

## Unverified dependencies

- Self-hosted metasearch instance.
- Email-finder and verification providers, their quotas and usage counters.
- Structured-output LLM for evidence-backed extraction.
- Sales store functions for pending-review leads and suppression checks.
- Owner notification channel and review view.

## Risk concerns

- Collection of third-party personal data (business contacts). Mitigated by business-data-only storage, provenance, retention limits for unreviewed leads, opt-in jurisdiction flags and the owner's review gate; lawful basis remains the owner's responsibility as controller.
- Website terms and robots rules. Mitigated by public-page-only, robots-honouring, rate-limited static fetches; social-network scraping is excluded.
- Hallucinated firmographics. Mitigated by mandatory evidence quotes and an independent rule-based fit recomputation.
- Metered spend. Mitigated by the quota pre-check, per-run cap and single retries.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The pipeline has only been observed in the reference implementation.

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
