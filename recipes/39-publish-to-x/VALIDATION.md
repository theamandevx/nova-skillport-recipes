# Validation report — Publish to X (`nova.content.publish-to-x` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- Current X pay-per-use prices, access tier, rate limits and image limits; price-table freshness window; daily post and spend caps (Sections E, Q).
- Whether a credit-balance read exists for X pay-per-use (Section C).
- `identity_provider` values are `TBD` or marked as UNVERIFIED X OAuth bindings in the manifest.

## Contradictions

None found. Checked: per-post approval vs scheduled posts (due posts trigger a reminder; publishing still needs approval); "never retry create" vs recovery (reconciliation only; unresolved → `publish_unknown` and a new approval); `with-media` failure vs fallback (no text-only degradation); already-published post (returns existing permalink as `completed`, takes no action).

## Unverified dependencies

- X API v2 create-post, media upload, user lookup, user posts and post lookup operations through registered adapters, with user-context OAuth 2.0 PKCE and the declared scopes.
- Operator-maintained price table with an `as_of` date.
- Content store supporting compare-and-set on the publish state.
- X's official weighted character-counting rules (library unresolved).

## Risk concerns

- Public, irreversible publication on the user's identity: mitigated by per-post approval bound to content hash, account and quote; read-back verification.
- Double posting: mitigated by the claim lock, a single `create` attempt, reconciliation by text hash, and `publish_unknown` instead of blind retries. Residual: X's eventual consistency could hide a just-created post during reconciliation — hence the human check.
- Cost surprise: mitigated by showing the quote (and the higher link price) before approval and blocking on a stale price table.
- Platform policy: automation rules and developer terms; the recipe publishes only the user's own approved posts and excludes replies, mentions and DMs.
- Over-broad OAuth scopes: mitigated by declaring the exact scope set and excluding message and follow scopes.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. The core flow runs in the reference implementation; not every control described here has evidence yet.

## §8 capability-register review checklist

- [x] Every capability has a stable ID, purpose, prerequisites, input/output contract and success criteria.
- [ ] Every callable tool has an implementation binding — **declared but UNVERIFIED** (marked as unresolved dependencies).
- [x] Every object/resource has a type, schema, ownership/tenant scope and permitted operations.
- [x] Every variant is enumerated with compatibility, constraints and explicit fallback behaviour (`text` and `with-media` are selected deterministically; no fallback between them).
- [ ] Provider, identity, consent, scopes, grants and bindings are enforced by the platform — **declared; enforcement depends on Skill Port / Vedaxi and X's OAuth**.
- [x] Dependencies, health and evidence are recorded with truthful status (`unverified`).
- [ ] The package cannot activate when mandatory checks fail — **depends on the platform's activation flow**.
- [x] AI-assisted repair cannot change security-sensitive controls (Section O).
- [x] Completion conditions and the result envelope are compatible with the capability contracts.
