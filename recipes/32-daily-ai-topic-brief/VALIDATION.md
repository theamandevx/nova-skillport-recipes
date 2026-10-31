# Validation report — Daily AI topic brief (`nova.knowledge.daily-ai-topic-brief` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform. The cross-provider fallback and failure-reporting design has no runtime evidence yet.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Skill Port's scheduler interface, and confirmation that it can host an independent watcher job (Section Q).
- How the platform identifies "provider" and "billing account" for the fallback-independence rule (the recipe defines the rule; the platform must supply the identifiers).
- Per-run and monthly token caps, run-record retention (Section E).
- Schema registry IDs: every `schema://` reference in the manifest is defined in Section D as a compact JSON Schema (cross-recipe reuse of the trending-topics-radar schemas is acknowledged in the text where it occurs), but none is registered with Skill Port yet.
- `identity_provider` for every tool in the manifest (`TBD`).

## Contradictions

None found. Checked: the fallback is automatic at run time vs "no silent fallback" (it is declared and approved at installation, policy-equivalent, used only on provider-side failures, and always disclosed in the brief and run record); the headlines-only path vs "fallback must satisfy the same constraints" (it is a deterministic template, not a model, carries no AI text, and is always paired with an alert); "one brief per day" vs a manual "send now" (counts as that day's brief unless the owner asks for another).

## Unverified dependencies

- Two model providers with separate accounts and credit balances in the target runtime.
- Provider credit/quota visibility (some providers expose none; recorded as `unknown`).
- The trending-topics radar recipe (or an equivalent topic store) for the subject keys.
- Owner chat channel, optional secondary channel, job registry and independent watcher.

## Risk concerns

- Reliability is the main risk: a scheduled job that fails silently. Mitigated by pre-flight checks, a mandatory independent fallback, a degraded template path, template alerts with no LLM dependency, a secondary alert channel and an independent watcher.
- A fallback that only looks independent (same router or billing account) shares the primary's failure modes; installation refuses it.
- Content risk is low (public items, owner-only), with line grounding and link provenance checks.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The daily digest itself has only been observed in the reference implementation; the outage test set (primary out of credit, both providers down, multi-day replay, scheduler miss, channel down) has no evidence and is mandatory before production.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour (primary → declared cross-provider fallback → deterministic headlines-only, all disclosed).
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow** (the recipe requires refusal when the fallback rule fails).
- [x] AI-assisted repair cannot change security-sensitive controls (Section O).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
