# Validation report — Action item follow-ups (`nova.calendar.action-item-follow-ups` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- A scheduler binding that fires per user at a local time (Section B; `TBD`).
- Retention period for nudge entries and storage/messaging cost budget (Sections E, Q).
- `identity_provider` for every tool in the manifest (`TBD`).

## Contradictions

None found. Checked: read-only scope vs the nudge log (the only write is an append to the recipe's own log, not to minutes or items); "skip silently" vs completion status (a correct skip is `completed` with `sent = false` and a reason); once-per-day rule vs retries (retries reconcile by message key first); "waiting on others" vs "never message others" (listed for the user only; chasing is handed to a comms recipe with approval).

## Unverified dependencies

- Minutes store with owner-tagged action items, and the optional `due_date` and `confidence` fields from recipe 25 (due-date grouping is unverified).
- Append-only nudge log with conditional append.
- Owner notification channel accepting plain text.
- Runtime scheduler with IANA time-zone support.

## Risk concerns

- Low overall risk: read-only and owner-only. Main risks are accuracy and annoyance, mitigated by the completeness and no-invention checks, the once-per-day claim, quiet hours and the silent empty skip.
- Misattribution: exact alias matching can miss nicknames (items fall into "owner unclear") or match a namesake; disclosed in Sections K and Q.
- Privacy: task text is personal data; it stays out of logs, metrics and operator views.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. Only the digest pattern (consolidate, skip when empty, owner-only plain text with a dashboard link) has been observed in the reference implementation. The rollback/compensation test is marked not applicable with a reason.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies; scheduler binding TBD).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour (default variants only; no channel fallback).
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls, including adding write scope or non-owner recipients (Section O).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
