# Validation report — Device heartbeat (`nova.ops.device-heartbeat` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present and every schema referenced in the manifest is defined inline in Section D. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- `identity_provider` for every tool in the manifest (`TBD`).
- Run-record retention period (Section E, `TBD`).
- The jobs-board contract (including the "stale renders unknown" rule) is declared as a dependency, not specified by Skill Port.

## Contradictions

None found. Checked: "alert once per transition" vs the outbox resend rule (resends reuse the same key; a possible duplicate is disclosed only where the channel cannot deduplicate); "never alert on first run" vs the corrupt-state rebaseline (the rebaseline sends one `state_reset` notice, not a device alert); the read-only fallback (`local-cli` → `control-api`) vs "no silent fallback" (both read-only, same schema, the switch is recorded).

## Unverified dependencies

- A device network adapter that exposes per-device online flag and last-seen time (mesh VPN local status command or control-plane API).
- A state store with atomic, versioned writes.
- An owner notification channel with delivery receipts (and ideally idempotent send).
- An optional jobs/health board that renders stale rows as `unknown`.

## Risk concerns

- Low risk overall: read-only and owner-only. The main residual risks are a silent monitor (mitigated by the source-failure notice and the stale-is-unknown board rule) and data leakage from the raw status output (mitigated by `additionalProperties:false` minimisation in S3).
- Mass-offline false alarms when the monitoring host itself loses the network (mitigated by the one-run hold in Section K; the threshold is a design default).
- A control-plane key with more than read scope would widen the blast radius; enablement must reject such keys.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The happy path and first-run baseline have only been observed in the reference implementation.

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
