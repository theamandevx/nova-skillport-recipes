# Validation report — Find free slots (`nova.calendar.find-free-slots` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Public-caller rate limit and per-client quota (Section E).
- Operator cost cap for calendar-API calls (Section E; no LLM is used).
- `identity_provider` for every tool in the manifest (`TBD`).
- Schema registry IDs: all schemas are defined inline in Section D but none is registered.

## Contradictions

None found. Checked: read-only recipe vs the optional busy-sync refresh (declared as Create / Update / Delete of the owner's own busy blocks, owner variant only, Allowed without approval because no third party is affected); public variant vs no-fallback rule (a public call never uses owner privileges); empty-slot outcome vs `completed` (allowed only with fresh sources and S7 confirmation).

## Unverified dependencies

- Calendar database with a slot-engine function and half-open interval tests.
- MCP server exposing `get_slots` and `get_availability`, with server-side stripping of overrides for public callers.
- Connected-calendar API access and mailbox metadata access for busy-sync.
- Runtime identity that distinguishes the owner from external callers.
- IANA time-zone database in the runtime.

## Risk concerns

- Double booking if a busy source is stale or an empty sync wipes blocks. Mitigated by freshness states, the empty-payload refusal, and the independent re-check in S7; recipe 22 also re-checks at booking time.
- Privacy: busy-block titles and sources could leak to outsiders. Mitigated by the public variant (free/busy only) and the output scan in Section I.
- Heuristic in-person detection may miss travel padding; disclosed in Section K.
- Daylight-saving errors; mitigated by IANA conversion and the round-trip check in S8.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The happy path, working hours that cross midnight, buffers, daily cap and travel padding have only been observed in the reference implementation.

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
