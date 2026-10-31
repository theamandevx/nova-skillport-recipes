# Validation report — Avatar video (`nova.content.avatar-video` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Per-video cap, daily budget, credit-to-currency table and maximum job age (Section E).
- Callback authentication-failure alert threshold and endpoint rate limit (Section H, S8).
- Whether failed jobs consume or refund credits (Section Q).
- `identity_provider` for every tool in the manifest (`TBD`, including the service identity used by the callback endpoint and reconciler).

## Contradictions

None found. Checked: callback and reconciler both completing the same job (monotonic, provider-job-keyed update makes it harmless); "no automatic resubmit" vs ambiguous submit (reconcile by correlation ID, then ask); interactive wait ending vs job still rendering (`partially_completed`, job keeps being reconciled); cost recorded at submit as an estimate vs actual credits (appended adjustment, never an edit).

## Unverified dependencies

- Avatar-video provider with async generation, status endpoint, catalogues and callbacks; whether it signs callbacks.
- A public HTTPS endpoint for callbacks under a restricted service identity (optional; reconciler-only supported).
- Scheduler for the reconciler with a single-flight lock.
- Private video bucket, video-record database function (job row plus usage row in one transaction), owner notification channel.

## Risk concerns

- Inbound endpoint: forged or replayed callbacks. Mitigated by signature or per-job hashed single-use token, constant-time comparison, rate limiting, alerts on repeated failures, and treating the body only as a trigger.
- Secret leakage: tokens embedded in URLs or workflow code can leak into logs or source control; mitigated by never logging tokens and redacting literal values, with rotation on exposure.
- Lost completions: mitigated by the reconciler and maximum job age.
- Expiring file links: mitigated by immediate download and re-fetching status for a fresh link.
- Likeness: avatars limited to stock or the user's own (created through the provider's consent flow).

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The core flow runs in the reference implementation; not every control described here has evidence yet.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour (callback and reconciler are concurrent, not fallbacks).
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O: callback authentication can never be weakened automatically).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
