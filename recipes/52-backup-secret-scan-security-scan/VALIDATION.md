# Validation report — Backups, secret scan and security scan (`nova.ops.backup-secret-scan-security-scan` 0.1.0, Draft)

## Summary

All 17 template sections (A–Q) are present and every schema referenced in the manifest is defined inline in Section D. The recipe is **ready for review, not for execution**: no capability is registered with Skill Port, no binding is confirmed on Vedaxi, and no Section P test has evidence on the target platform.

## Missing fields

- Skill Port publisher ID and assigned recipe ID (Section A).
- Registered binding IDs, schema registry IDs and version ranges for every operation (Section B; marked `UNVERIFIED` / `TBD`).
- `identity_provider` for every tool in the manifest (`TBD`).
- Off-site provider and storage cost (Section E, `TBD`).
- The `backup:drill` scope is declared in the manifest but its exact grant model (an operator-supervised session) depends on the platform.

## Contradictions

None found. Checked: "never fall back to plaintext" vs the off-site optionality (off-site may be absent, but local archives are always encrypted, and its absence is reported as `partially_completed`); `sqlite-online → cold-stop` fallback vs "no silent fallback" (allowed only for `cold_ok` items in the approved window and recorded per item); "retention deletes archives" vs "off-site objects never deleted" (prune touches local archives only).

## Unverified dependencies

- SQLite online backup access to live databases through read-only connections.
- Authenticated public-key encryption tool and off-host private-key custody.
- S3-compatible storage with versioning or object lock and a write-only credential (optional).
- Private git remote with push protection, a repository-scoped deploy key, gitleaks hooks.
- Trivy with a reachable vulnerability database; images pinned by digest.
- Owner channel and optional jobs board.

## Risk concerns

- Archives contain the most sensitive state: mitigated by mandatory authenticated encryption, keys apart from archives, private key off the host, 0600 permissions and no plaintext intermediates.
- Secret leakage through the mirror: mitigated by whitelist + deny list, credential-stripped exports, redaction, gitleaks in the script and in pre-commit and pre-push hooks, remote push protection, and "scanner missing = blocked". Residual: novel secret formats (Section Q).
- Credential exposure through process listings or error messages: mitigated by environment-only passwords, sanitised timeout errors, refusal of unparseable connection strings without echo, and `hidepid`.
- Losing the off-host private key makes every archive unrecoverable; custody is a human process.
- Cold-stop causes brief downtime; limited to approved items, window and duration, with a health check.

## Tests still needed

Every row of Section P needs evidence on Skill Port / Vedaxi. Encrypted backups, the secret-scan-gated mirror, the scheduled scan and the restore dry run have been observed in the reference implementation; the remaining controls still need first evidence.

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
