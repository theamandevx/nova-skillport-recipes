# Validation report — Payment reminders (`nova.finance.payment-reminders` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**. No capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform. Catalogue status is Built: the behaviour has run in Nova's reference implementation. That is historical observation, not a current test run, and not every control described here has evidence yet (Section P lists the tests still needed).

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Schemas: none missing — every `schema://` reference in the manifest is defined as a JSON Schema with a matching `$id` in Section D; reused sibling-recipe schemas are acknowledged there.
- Whether the target channel supports client message keys (affects duplicate prevention after an ambiguous send).
- `identity_provider` for every tool in the manifest (`TBD`).

## Contradictions

None found. Checked: the quiet-hours bypass against the "never during quiet hours" soft goal (the bypass is explicit and user-configurable); the one-shot rule against the user re-send request (re-send clears one flag with audit); at-least-once delivery against "each sent once" (a single duplicate after an ambiguous outcome is disclosed, never hidden).

## Unverified dependencies

- Obligation store with `alert_scan`, `alert_mark` and `advance_cycle` dispatcher functions.
- Owner channel adapter returning delivery receipts.
- Runtime lock, kill switch and per-user time zone.
- Upstream: `nova.finance.upcoming-payments-radar`; optional: `nova.finance.auto-mark-bills-paid` for one-off auto-charge rows past due.

## Risk concerns

- Missed payment if the channel silently fails. This is mitigated by send-then-confirm (unsent stages stay queued) and escalation after 3 failed runs.
- Notification fatigue. This is mitigated by one digest per run, supersession and quiet hours.
- Untrusted merchant strings from emails. These are escaped, and no URLs are ever included.
- Auto-charge items that failed to charge are not detected (Section Q).

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The core flow has run in the reference implementation; not every control described here has evidence yet.

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
