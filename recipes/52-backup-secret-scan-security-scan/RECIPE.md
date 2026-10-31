# Backups, secret scan and security scan

## A. Identity and ownership

- **Recipe ID:** `nova.ops.backup-secret-scan-security-scan` (proposed until assigned by Skill Port)
- **Name:** Backups, secret scan and security scan
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** security
- **Tags:** encrypted-backups, sqlite-wal, restore-drill, retention, config-mirror, gitleaks, trivy, credential-hygiene
- **Summary:** Nightly, the agent takes consistent, encrypted backups of the assistant's state and the automation hub, applies retention and (recommended) copies them off-site, and mirrors whitelisted configuration one way into a private git repository behind a secret scanner that blocks any push containing a secret. Weekly, it scans running container images for vulnerabilities and alerts on any HIGH or CRITICAL finding. A monthly restore drill proves the backups work, and database passwords never appear on a command line.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** A host scheduler; SQLite (online backup API) for the assistant session store and the automation hub; an authenticated-encryption tool with public-key recipients (for example age); local storage; optionally S3-compatible object storage with versioning or object lock; git with a private remote and a repository-scoped deploy key; gitleaks; Trivy; an owner notification channel; optionally a jobs/health board. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — encrypted nightly backups, a tested restore dry run, a secret-scan-gated configuration mirror and a scheduled image vulnerability scan run in Nova's reference implementation; not every control described here has evidence yet (Section P lists the tests still needed). Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.backup.snapshot` | Capability | Take consistent point-in-time copies | medium | UNVERIFIED |
| `nova.backup.snapshot.create` | Operation (read source, create temp copy) | Snapshot one backup item | medium | UNVERIFIED |
| `nova.backup.verify` | Capability | Check snapshot integrity | low | UNVERIFIED |
| `nova.backup.verify.integrity` | Operation (read) | Run integrity checks on a snapshot | low | UNVERIFIED |
| `nova.backup.seal` | Capability | Encrypt and checksum | high | UNVERIFIED |
| `nova.backup.seal.encrypt` | Operation (create) | Encrypt to the recipient public key and write a checksum manifest | high | UNVERIFIED |
| `nova.backup.store` | Capability | Keep archives locally and off-site | medium | UNVERIFIED |
| `nova.backup.store.write_local` | Operation (create) | Write the sealed archive to the local backup directory | medium | UNVERIFIED |
| `nova.backup.store.copy_offsite` | Operation (create, external) | Upload to write-only off-site storage | medium | UNVERIFIED |
| `nova.backup.retention` | Capability | Apply the retention policy | medium | UNVERIFIED |
| `nova.backup.retention.prune` | Operation (delete, policy-bound) | Delete local archives beyond the policy | medium | UNVERIFIED |
| `nova.backup.drill` | Capability | Prove restorability | medium | UNVERIFIED |
| `nova.backup.drill.run` | Operation (create in scratch only) | Restore into an isolated scratch environment and verify | medium | UNVERIFIED |
| `nova.mirror.config` | Capability | One-way mirror of whitelisted config to a private repo | high | UNVERIFIED |
| `nova.mirror.config.collect` | Operation (read live, update working tree) | Copy whitelisted files, export workflows without credentials, redact | high | UNVERIFIED |
| `nova.mirror.config.scan` | Operation (read) | Run gitleaks on the staged changes | high | UNVERIFIED |
| `nova.mirror.config.commit_push` | Operation (create, external) | Commit and push when the scan is clean | high | UNVERIFIED |
| `nova.security.vuln_scan` | Capability | Scan running images | low | UNVERIFIED |
| `nova.security.vuln_scan.scan_images` | Operation (read) | Trivy scan per image digest, HIGH/CRITICAL | low | UNVERIFIED |
| `nova.notify.owner` | Capability | Deliver a message to the owner's own channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Send failures, findings and staleness alerts | low | UNVERIFIED |
| `nova.jobs.registry` | Capability | Show job health on the jobs board | low | UNVERIFIED |
| `nova.jobs.registry.report` | Operation (update) | Upsert this recipe's job rows | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.backup.snapshot.create.sqlite-online` — **default for every SQLite database** (assistant session store, automation hub). Uses SQLite's online backup API (or `VACUUM INTO`) through a read-only connection, which yields a consistent copy even while the database runs in WAL mode. Copying the live `.db` file (with or without its `-wal` file) is never used.
- `nova.backup.snapshot.create.cold-stop` — briefly stop one service, copy its data, restart it and run its health check. Eligible only for items flagged `cold_ok`, inside the maintenance window approved at enablement, for at most 2 minutes. Fallback `sqlite-online → cold-stop` is allowed only under those conditions and is recorded.
- `nova.backup.snapshot.create.file-copy` — plain files (configuration, skills, compose and unit files). Never used for databases; there is no fallback from either database variant to `file-copy`.
- `nova.backup.store.copy_offsite.object-lock` — recommended: S3-compatible storage with versioning or object lock, using a credential that can write but not delete or overwrite. Optional; when not configured, the run reports "no off-site copy" every time instead of silently passing.
- All other operations — only the default variant is supported.

**Bindings:** `binding://nova/backup/snapshot`, `binding://nova/backup/verify`, `binding://nova/backup/seal`, `binding://nova/backup/store-local`, `binding://nova/backup/store-offsite`, `binding://nova/backup/prune`, `binding://nova/backup/drill`, `binding://nova/mirror/collect`, `binding://nova/mirror/scan`, `binding://nova/mirror/push`, `binding://nova/security/trivy`, `binding://nova/notify/owner`, `binding://nova/jobs/report` — all `status: UNVERIFIED`. Reference implementation: shell and Python scripts on systemd timers, a gitleaks container run against the mirror working tree plus git hooks, and a Trivy container run against the images of the running stack.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| Backup item | `schema://nova/backup-secret-scan-security-scan/backup-item@0.1.0` | operator-declared | read |
| Snapshot (temporary) | part of `backup-run@0.1.0` | this run | create, delete after sealing |
| Sealed archive + checksum manifest | `schema://nova/backup-secret-scan-security-scan/archive@0.1.0` | the owner's backup store | create, read; delete by retention only |
| Mirror working tree / commit | `schema://nova/backup-secret-scan-security-scan/mirror-run@0.1.0` | one private repository | update (whitelisted paths), create (commit) |
| Scan report | `schema://nova/backup-secret-scan-security-scan/vuln-report@0.1.0` | the owner's tenant | create |
| Drill report | `schema://nova/backup-secret-scan-security-scan/drill-report@0.1.0` | the owner's tenant | create |

**Authorization requirements (declared, never a grant):** `backup:read-sources` (read-only access to listed items), `backup:write` (local backup directory only), `backup:offsite:write` (no delete or overwrite), `backup:prune` (local archives created by this recipe only), `service:restart` (cold-stop items only, in the window), `backup:drill` (operator-supervised drill session, scratch only), `mirror:write` (one repository, via a repository-scoped deploy key — not a personal token), `containers:inspect` (list images and digests, read-only), `notify:owner`, `jobs:report`. Secrets by reference only: `secret_ref://backup-recipient-public-key` (public; the private key stays off the host with the owner), `secret_ref://offsite-write-only`, `secret_ref://mirror-deploy-key`, `secret_ref://db-readonly` (schema snapshot, environment-only).

**Policy / trust:** risk class `high` — archives contain the assistant's most sensitive state. Mandatory controls: encryption before anything leaves the snapshot step; keys stored apart from archives; secret scanning before every push; credentials never in process arguments; images pinned by digest so a scan describes what actually runs.

## C. Intent and applicability

- **User problem:** A self-hosted assistant holds months of configuration, memory and automations on one server. A disk failure, a bad upgrade or a mistake could erase it, a careless config backup could leak secrets into a repository, and unpatched images quietly accumulate known vulnerabilities.
- **Desired outcome:** Restorable encrypted backups that are proven by drills, a reviewable history of configuration with no secrets in it, a weekly honest picture of image vulnerabilities, and alerts only when something needs attention.
- **Trigger or entry conditions:** Scheduled: backups and mirror nightly (default 03:30 and 04:00 owner time), vulnerability scan weekly (default Sunday 05:00), restore drill monthly (human-assisted). On demand: "back up now" runs the backup path.
- **Applicable situations:** A single-host or small self-hosted assistant stack with SQLite stores and containers.
- **Out-of-scope situations:** Backing up managed cloud databases (use the provider's point-in-time recovery; this recipe mirrors only a schema snapshot without data); restoring into production (an owner-run procedure, not autonomous); upgrading images or patching (deliberate, approved changes); scanning third-party systems; rewriting git history.
- **Required inputs:** `backup-config@0.1.0` (see Section D).
- **Prerequisites:** Encryption recipient public key installed and the private key held off the host; backup directory on a different disk or volume where possible; private repository with push protection and a deploy key; gitleaks hooks installed in the mirror repository; Trivy with network access to its vulnerability database; owner channel.
- **Required capabilities:** all objects in Section B (proposed, UNVERIFIED); off-site storage optional.
- **Unsupported or missing capabilities:** None registered on Skill Port; each binding is a dependency to resolve before activation.
- **Assumptions:** Verified in the reference implementation: encrypted nightly archives, a restore dry run, the secret-scan-gated mirror, scheduled image scans. Not every control in this recipe has evidence yet; all of them are mandatory before activation. Assumed: the target host allows a read-only SQLite connection to live databases.
- **Ambiguity policy:** Stop on anything security-relevant (missing key, scanner unavailable, unknown file in a whitelisted directory that matches a deny pattern): skip that part, never proceed unencrypted or unscanned, and report.
- **Vedaxi task type:** Not applicable — this recipe does not map to any of the seven task types Vedaxi supports today.

## D. Inputs and outputs

**Inputs** (operator configuration, approved at enablement)

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `items` | array of `backup-item@0.1.0` | yes | operator | — | empty → `blocked` |
| `retention` | `{daily, weekly, monthly, keep_min}` | no | operator | 7 / 4 / 3, `keep_min` 3 | `keep_min` < 2 → raised to 2 |
| `offsite` | object or `null` | no | operator | `null` | null → reported every run as "no off-site copy" |
| `mirror` | `mirror-config@0.1.0` | no | operator | disabled | invalid → mirror path `blocked` |
| `scan_images` | `"running"` or array of image references pinned by digest | no | operator | `"running"` | unpinned tag → scanned but flagged |
| `stale_after_hours` | integer 24–72 | no | operator | 26 | clamp |

```json
{"$id":"schema://nova/backup-secret-scan-security-scan/backup-config@0.1.0","type":"object","required":["items"],"additionalProperties":false,"properties":{"items":{"type":"array","minItems":1,"items":{"$ref":"schema://nova/backup-secret-scan-security-scan/backup-item@0.1.0"}},"retention":{"type":"object","properties":{"daily":{"type":"integer","minimum":1,"default":7},"weekly":{"type":"integer","minimum":0,"default":4},"monthly":{"type":"integer","minimum":0,"default":3},"keep_min":{"type":"integer","minimum":2,"default":3}}},
 "offsite":{"type":["object","null"],"properties":{"endpoint_ref":{"type":"string"},"bucket_ref":{"type":"string"},"credential_ref":{"type":"string","pattern":"^secret_ref://"},"object_lock":{"type":"boolean"}}},
 "mirror":{"$ref":"schema://nova/backup-secret-scan-security-scan/mirror-config@0.1.0"},"scan_images":{"oneOf":[{"const":"running"},{"type":"array","items":{"type":"string","pattern":"@sha256:[a-f0-9]{64}$"}}]},"stale_after_hours":{"type":"integer","minimum":24,"maximum":72,"default":26}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `backup_run` | `backup-run@0.1.0` | `run_id`, `status`, `archives`, `offsite`, `pruned` | run record, jobs board | low (no contents) | Section I |
| `mirror_run` | `mirror-run@0.1.0` | `run_id`, `status`, `scan`, `pushed` | run record, jobs board | low (rule IDs and paths, never secret values) | Section I |
| `vuln_report` | `vuln-report@0.1.0` | `run_id`, `status`, `db_fresh`, `images` | run record, owner alert | low | Section I |
| `drill_report` | `drill-report@0.1.0` | `drill_id`, `result`, `checks` | run record | low | Section I |

```json
{"$id":"schema://nova/backup-secret-scan-security-scan/backup-run@0.1.0","type":"object","required":["run_id","status","archives","offsite","pruned"],
 "properties":{"run_id":{"type":"string"},"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},
 "archives":{"type":"array","items":{"$ref":"schema://nova/backup-secret-scan-security-scan/archive@0.1.0"}},
 "offsite":{"enum":["copied","not_configured","failed"]},"pruned":{"type":"array","items":{"type":"string"}},
 "variant_by_item":{"type":"object","additionalProperties":{"enum":["sqlite-online","cold-stop","file-copy"]}},
 "last_good_at":{"type":["string","null"],"format":"date-time"},"warnings":{"type":"array","items":{"type":"string"}}}}
```

**Other schemas** (compact JSON Schema):

```json
{"$id":"schema://nova/backup-secret-scan-security-scan/backup-item@0.1.0","type":"object","required":["item_id","kind","source_ref"],"properties":{"item_id":{"type":"string","pattern":"^[a-z0-9-]{2,40}$"},"kind":{"enum":["sqlite","files"]},"source_ref":{"type":"string","description":"logical reference resolved by the runtime, never a user path"},"service":{"type":["string","null"]},"cold_ok":{"type":"boolean","default":false},"include":{"type":"array","items":{"type":"string"}},"exclude":{"type":"array","items":{"type":"string"}}}}
{"$id":"schema://nova/backup-secret-scan-security-scan/snapshot@0.1.0","type":"object","required":["item_id","variant","taken_at","temp_ref"],"properties":{"item_id":{"type":"string"},"variant":{"enum":["sqlite-online","cold-stop","file-copy"]},"taken_at":{"type":"string","format":"date-time"},"temp_ref":{"type":"string"},"bytes":{"type":"integer"},"integrity":{"enum":["ok","failed","not_applicable"]}}}
{"$id":"schema://nova/backup-secret-scan-security-scan/archive@0.1.0","type":"object","required":["archive_id","created_at","items","sha256","encrypted","recipient_fingerprint"],"properties":{"archive_id":{"type":"string"},"created_at":{"type":"string","format":"date-time"},"items":{"type":"array","items":{"type":"string"}},"bytes":{"type":"integer"},"sha256":{"type":"string","pattern":"^[a-f0-9]{64}$"},"encrypted":{"const":true},"recipient_fingerprint":{"type":"string"},"tier":{"enum":["daily","weekly","monthly"]},"drilled_at":{"type":["string","null"],"format":"date-time"}}}
{"$id":"schema://nova/backup-secret-scan-security-scan/offsite-result@0.1.0","type":"object","required":["archive_id","state"],"properties":{"archive_id":{"type":"string"},"state":{"enum":["copied","not_configured","failed"]},"remote_sha256":{"type":["string","null"]},"retained_until":{"type":["string","null"],"format":"date-time"}}}
{"$id":"schema://nova/backup-secret-scan-security-scan/prune-plan@0.1.0","type":"object","required":["keep","delete"],"properties":{"keep":{"type":"array","items":{"type":"string"}},"delete":{"type":"array","items":{"type":"string"}},"newest_good":{"type":"string"}}}
{"$id":"schema://nova/backup-secret-scan-security-scan/mirror-config@0.1.0","type":"object","required":["enabled","include","deny"],"properties":{"enabled":{"type":"boolean"},"include":{"type":"array","items":{"type":"string"}},"deny":{"type":"array","items":{"type":"string"},"default":[".env","*.key","*.pem","*credential*","*secret*","*token*","auth*.json","sessions/**"]},"exports":{"type":"array","items":{"enum":["workflows_without_credentials","schema_snapshot_without_data","unit_files"]}},"redact_patterns":{"type":"array","items":{"type":"string"}},"repo_ref":{"type":"string"},"deploy_key_ref":{"type":"string","pattern":"^secret_ref://"}}}
{"$id":"schema://nova/backup-secret-scan-security-scan/mirror-run@0.1.0","type":"object","required":["run_id","status","scan","pushed"],"properties":{"run_id":{"type":"string"},"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},"changed_files":{"type":"integer"},"redactions":{"type":"integer"},"scan":{"$ref":"schema://nova/backup-secret-scan-security-scan/scan-result@0.1.0"},"pushed":{"type":"boolean"},"commit":{"type":["string","null"]}}}
{"$id":"schema://nova/backup-secret-scan-security-scan/scan-result@0.1.0","type":"object","required":["tool","clean","findings"],"properties":{"tool":{"const":"gitleaks"},"clean":{"type":"boolean"},"findings":{"type":"array","items":{"type":"object","required":["rule_id","file","line"],"properties":{"rule_id":{"type":"string"},"file":{"type":"string"},"line":{"type":"integer"}}}}}}
{"$id":"schema://nova/backup-secret-scan-security-scan/vuln-report@0.1.0","type":"object","required":["run_id","status","db_fresh","images"],"properties":{"run_id":{"type":"string"},"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},"db_fresh":{"type":"boolean"},"images":{"type":"array","items":{"type":"object","required":["image","digest_pinned","critical","high"],"properties":{"image":{"type":"string"},"digest_pinned":{"type":"boolean"},"critical":{"type":"integer"},"high":{"type":"integer"},"fixable":{"type":"integer"},"new_since_last":{"type":"integer"}}}}}}
{"$id":"schema://nova/backup-secret-scan-security-scan/drill-request@0.1.0","type":"object","required":["archive_id","scratch_ref"],"properties":{"archive_id":{"type":"string"},"scratch_ref":{"type":"string"},"checks":{"type":"array","items":{"enum":["decrypt","checksum","sqlite_integrity","row_counts","app_boot","workflow_count"]}}}}
{"$id":"schema://nova/backup-secret-scan-security-scan/drill-report@0.1.0","type":"object","required":["drill_id","archive_id","result","checks"],"properties":{"drill_id":{"type":"string"},"archive_id":{"type":"string"},"result":{"enum":["passed","failed","not_run"]},"checks":{"type":"object","additionalProperties":{"enum":["pass","fail","skipped"]}},"performed_by":{"type":"string"},"at":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/backup-secret-scan-security-scan/alert@0.1.0","type":"object","required":["kind","text","idempotency_key"],"properties":{"kind":{"enum":["backup_failed","backup_stale","offsite_failed","secret_detected","mirror_stale","vuln_found","scan_db_stale","drill_failed","drill_due"]},"text":{"type":"string","maxLength":800},"idempotency_key":{"type":"string"}}}
{"$id":"schema://nova/backup-secret-scan-security-scan/delivery@0.1.0","type":"object","required":["idempotency_key","delivered"],"properties":{"idempotency_key":{"type":"string"},"delivered":{"type":"boolean"},"message_id":{"type":["string","null"]}}}
{"$id":"schema://nova/backup-secret-scan-security-scan/job-report@0.1.0","type":"object","required":["job_key","state","reported_at"],"properties":{"job_key":{"enum":["backup","mirror","vuln-scan","restore-drill"]},"state":{"enum":["ok","late","failing","unknown"]},"last_success":{"type":["string","null"],"format":"date-time"},"reported_at":{"type":"string","format":"date-time"}}}
```

Notes: `scan-result@0.1.0`: findings never carry the matched value.

**Example (fictional)**

Config: two items — `assistant-state` (files: config, skills, memory) and `hub-db` (sqlite, service `automation-hub`, `cold_ok: false`); retention 7/4/3; off-site configured with object lock; mirror enabled. `backup_run`: `{"run_id":"bk_1031","status":"completed","archives":[{"archive_id":"2026-10-31-daily","created_at":"2026-10-31T07:31:12Z","items":["assistant-state","hub-db"],"bytes":18432000,"sha256":"<64 hex>","encrypted":true,"recipient_fingerprint":"age1…f3k","tier":"daily","drilled_at":null}],"offsite":"copied","pruned":["2026-10-23-daily"],"variant_by_item":{"assistant-state":"file-copy","hub-db":"sqlite-online"},"last_good_at":"2026-10-31T07:31:12Z"}`

`mirror_run` with a planted test token: `{"run_id":"mr_1031","status":"blocked","changed_files":3,"redactions":0,"scan":{"tool":"gitleaks","clean":false,"findings":[{"rule_id":"generic-api-key","file":"skills/example/config.yaml","line":12}]},"pushed":false,"commit":null}` → alert: "Config mirror blocked: possible secret (generic-api-key) in skills/example/config.yaml line 12. Nothing was pushed."

## E. Constraints and budgets

- **Hard constraints:** nothing is written to the backup store or sent off-site unencrypted; encryption is authenticated and to a public-key recipient whose private key is not on the host; archives and keys are stored apart; SQLite databases are snapshotted only through the online backup API (or cold-stop in the approved window), never by copying live files; a snapshot that fails its integrity check is never sealed or counted as good; retention never deletes the newest good archive or drops below `keep_min` good archives, and never touches off-site copies; the mirror is one way (live → repository; never pulls or applies from the repository), copies only whitelisted paths, excludes deny patterns, and exports workflows without credentials; **nothing is pushed unless gitleaks ran and was clean** (scanner missing or erroring = blocked); allowlisting a finding needs owner approval; findings never echo the secret value; images are scanned by digest and a scan with a stale vulnerability database is never reported as clean; **database passwords and connection strings never appear in process arguments, logs or error messages** — they are passed through environment variables or `0600` credential files, command-line timeout errors are re-raised without the command line, connection strings with unescaped reserved characters in the credentials are refused without echoing any part of them, and `/proc` should be mounted with `hidepid` on multi-user hosts.
- **Soft constraints:** backups finish before the mirror runs; alerts consolidated per job per night.
- **Resource and tenant scope:** the declared items, one backup store, one mirror repository, the running images of this stack.
- **Data sources allowed:** the declared items; the container runtime's image list.
- **External transfers permitted:** encrypted archives to the configured off-site store; whitelisted, scanned config to the private repository; vulnerability-database downloads by Trivy; alerts to the owner channel.
- **Time limit:** backup ≤ 30 minutes; mirror ≤ 10 minutes; scan ≤ 30 minutes; cold-stop downtime ≤ 2 minutes per service.
- **Cost or token budget:** no LLM is used; off-site storage cost `TBD` (operator).
- **Maximum steps:** 11 (S1–S11), split across schedules.
- **Maximum retries and correction cycles:** snapshot 1 retry per item; off-site upload 2 retries; push 2 retries (transport errors only); scan 1 retry; correction cycles 1.
- **Rate limits and concurrency:** one run per job at a time (lock); backup and mirror never overlap.
- **Retention and logging constraints:** archives per `retention`; run records 90 days; logs keep sizes, hashes, rule IDs and file paths — never contents, secrets or connection strings.
- **Prohibited actions:** restoring into production; rewriting git history; force-pushing; auto-allowlisting findings; auto-upgrading images; deleting off-site objects; printing any credential.
- **Stop conditions:** recipient key missing; free disk below twice the expected archive size; scanner unavailable; lock held.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create, then Delete | Temporary snapshots (deleted after sealing or on failure) | preflight passed | Allowed | integrity check; temp directory empty at the end |
| Create | Sealed archive + checksum manifest | snapshot integrity `ok` | Allowed | checksum and header check |
| Create | Off-site object | archive sealed; write-only credential | Allowed | remote checksum equals local |
| Create | Mirror commit and push | scan clean; whitelisted paths only | Allowed (scope approved at enablement) | remote ref equals local commit |
| Create | Scan and drill reports | — | Allowed | schema-valid |
| Read | Declared items (read-only connections), image list, repository state | — | Allowed | — |
| Update | Mirror working tree (whitelisted paths) | — | Allowed | diff limited to whitelist |
| Update | Service state (stop and start) for `cold_ok` items | inside the approved window | Approval Required (window, at enablement) | health check passes after restart |
| Update | Jobs board rows | — | Allowed | `reported_at` = run time |
| Delete | Local archives beyond retention | newest good and `keep_min` protected | Allowed within the approved policy | prune plan recorded |
| Delete | Off-site objects; git history | — | Not Allowed | — |
| External side effect | push to the private repository; off-site upload; owner alerts | as above | Allowed | receipts |

- **Idempotency:** archive ID = date + tier; re-running the same night overwrites nothing (a second run produces a suffixed ID and the prune plan keeps both until the next cycle). Pushes are fast-forward only. Alerts keyed by job + night + kind.
- **Expected side effects:** new archive locally (and off-site); a commit in the mirror when config changed; owner messages only on failure, staleness, secrets or HIGH/CRITICAL findings.
- **Compensation:** a failed partial upload is aborted (multipart abort) and marked failed; a temp snapshot is deleted; a service stopped for cold-stop is restarted even if the copy fails.
- **Irreversible:** pruned local archives; a pushed commit (if a secret ever reaches the remote despite the scanner, the secret is rotated — deleting the commit is not enough).
- **Ambiguous outcomes:** an upload timeout is resolved by comparing the remote checksum before retrying; a push timeout by reading the remote ref.

## G. Autonomy and approval policy

- **Default autonomy level:** A2 — bounded autonomy for scheduled backup, mirror and scan.
- **Planning level:** A2.
- **Execution level:** A2 within the enablement scope; A1 for cold-stop (pre-approved window only).
- **Verification level:** A2.
- **Recovery level:** A2 for bounded retries; A0 for anything involving secrets found, allowlists or restores.
- **Approval required for:** enabling the recipe (items, retention, off-site target, mirror whitelist and repository, cold-stop window); changing the whitelist or deny list; adding a gitleaks allowlist entry; running a restore drill (operator supplies the private key in scratch).
- **Actions never permitted:** unencrypted archives; pushing unscanned content; allowlisting to unblock a push without the owner; restoring into production; force-pushing; deleting off-site objects; upgrading images automatically; passing credentials on a command line.
- **Conditions requiring clarification:** a whitelisted directory contains a file matching a deny pattern (excluded and reported for the owner to decide).
- **Conditions requiring escalation:** secret detected; backup failed or stale; drill failed or overdue; HIGH/CRITICAL findings; vulnerability database stale.
- **Safe state after stop, cancellation or escalation:** temp snapshots deleted, services running, nothing pushed, no partial archive marked good, previous good archives intact.

## H. Execution plan

**S1 — Preflight**
1. Objective: start only when the run can finish safely.
2. Capability: runtime lock; local checks.
3. Input mapping: `backup-config@0.1.0`.
4. Dependencies: none.
5. Preconditions: recipe enabled.
6. Action / side effects: take the job lock; validate config; confirm the recipient public key is present by reference; confirm free disk ≥ 2 × the last archive size; confirm the time window.
7. Autonomy: A2.
8. Timeout / retry: 10 s / none.
9. Idempotency: not applicable.
10. Postconditions: lock held; config valid.
11. Verification: every check recorded pass/fail.
12. Failure routes: key missing or disk low → `blocked`; lock held → `cancelled` (`concurrent_run`).
13. Escalation triggers: blocked → `backup_failed` alert.
14. Evidence: check results.

**S2 — Take consistent snapshots**
1. Objective: a consistent copy of every item.
2. Capability: `nova.backup.snapshot.create` (variant per item kind).
3. Input mapping: each `backup-item` → `snapshot@0.1.0`.
4. Dependencies: S1.
5. Preconditions: item reachable; for cold-stop, inside the window.
6. Action / side effects: SQLite → online backup API through a read-only connection into the temp directory (mode `0700`); files → copy with includes/excludes; cold-stop → stop, copy, start, health check.
7. Autonomy: A2 (A1 for cold-stop within the approved window).
8. Timeout / retry: 10 minutes per item / 1 retry.
9. Idempotency: temp paths per run ID.
10. Postconditions: one snapshot per item.
11. Verification: snapshot exists with non-zero size; services healthy after any cold-stop.
12. Failure routes: an item fails → archive the others, run `partially_completed`; a service fails its health check after restart → restart once more, then escalate immediately.
13. Escalation triggers: service unhealthy after cold-stop.
14. Evidence: variant per item, sizes, timestamps.

**S3 — Verify snapshot integrity**
1. Objective: never seal a broken copy.
2. Capability: `nova.backup.verify.integrity`.
3. Input mapping: snapshots.
4. Dependencies: S2.
5. Preconditions: snapshots exist.
6. Action / side effects: SQLite `PRAGMA integrity_check` (and a row count of key tables); file trees: entry count and size compared with the previous run (a drop over 50 % is flagged).
7. Autonomy: A2.
8. Timeout / retry: 5 minutes / none.
9. Idempotency: read-only.
10. Postconditions: integrity `ok` per item, or `failed`.
11. Verification: `integrity_check` returns "ok".
12. Failure routes: failed item → excluded from the archive; run `partially_completed` with a `backup_failed` alert for that item.
13. Escalation triggers: the same item failing two nights in a row.
14. Evidence: integrity results, counts.

**S4 — Seal, store and copy off-site**
1. Objective: an encrypted, checksummed archive at rest locally and, when configured, off-site.
2. Capability: `nova.backup.seal.encrypt`; `nova.backup.store.write_local`; `nova.backup.store.copy_offsite.object-lock` (optional).
3. Input mapping: verified snapshots + recipient public key → `archive@0.1.0`; archive + manifest + `offsite` config → `offsite-result@0.1.0`.
4. Dependencies: S3.
5. Preconditions: at least one verified snapshot; off-site configured (otherwise `not_configured` is recorded).
6. Action / side effects: stream tar → authenticated encryption to the recipient (no plaintext archive is ever written); write the SHA-256 manifest; write to the backup directory with mode `0600`; delete temp snapshots; upload with the write-only credential (object lock or versioning retains it per policy).
7. Autonomy: A2.
8. Timeout / retry: seal 15 minutes / 1 retry; upload 20 minutes / 2 retries after checking the remote checksum.
9. Idempotency: archive ID per night and tier; off-site object key = archive ID, and the credential cannot overwrite.
10. Postconditions: sealed archive + manifest; temp directory empty; `offsite-result@0.1.0`.
11. Verification: checksum recomputed; the file starts with the encryption format header; no plaintext file exists in the backup directory; remote checksum equals local.
12. Failure routes: encryption fails → delete partial output, `failed`, never fall back to plaintext; upload fails after retries → abort any partial multipart upload, `offsite_failed` alert, run `partially_completed`.
13. Escalation triggers: `failed` → alert; off-site failing 3 nights running → alert.
14. Evidence: archive ID, size, checksum, recipient fingerprint, remote checksum, retention date.

**S5 — Apply retention**
1. Objective: bounded local storage without risking the last good copies.
2. Capability: `nova.backup.retention.prune`.
3. Input mapping: local archive list, `retention`.
4. Dependencies: S4 succeeded tonight (otherwise skip pruning).
5. Preconditions: tonight's archive verified.
6. Action / side effects: compute `prune-plan@0.1.0` (daily/weekly/monthly tiers); protect the newest good archive and at least `keep_min` good archives; delete only archives this recipe created.
7. Autonomy: A2 within the approved policy.
8. Timeout / retry: 2 minutes / none.
9. Idempotency: re-running deletes nothing new.
10. Postconditions: plan applied.
11. Verification: remaining good archives ≥ `keep_min`; newest good present.
12. Failure routes: plan would violate a protection → skip pruning, warn.
13. Escalation triggers: disk still below threshold after pruning → alert.
14. Evidence: prune plan.

**S6 — Collect mirror changes**
1. Objective: an up-to-date, secret-free copy of configuration in the working tree.
2. Capability: `nova.mirror.config.collect`.
3. Input mapping: `mirror-config@0.1.0`.
4. Dependencies: S4 finished (mirror runs after backup).
5. Preconditions: mirror enabled; working tree clean or only this recipe's changes.
6. Action / side effects: copy whitelisted paths; skip deny patterns; export workflows with credentials stripped; regenerate the schema snapshot without data using a read-only database credential passed through the environment; redact known token shapes (bearer tokens, JWT-shaped strings, provider key prefixes) in exports.
7. Autonomy: A2.
8. Timeout / retry: 5 minutes / 1 retry.
9. Idempotency: deterministic for the same live state.
10. Postconditions: staged diff limited to whitelisted paths.
11. Verification: every changed path matches the whitelist and no deny pattern; no process during the step had a password in its arguments.
12. Failure routes: a deny-pattern file inside a whitelisted directory → excluded and reported; export fails → mirror `partially_completed` (other files still scanned and pushed).
13. Escalation triggers: deny-pattern exclusions appearing → owner told once per file.
14. Evidence: changed-file count, redaction count, exclusions (paths only).

**S7 — Secret scan**
1. Objective: no secret ever leaves in a commit.
2. Capability: `nova.mirror.config.scan` (gitleaks with the repository's configuration).
3. Input mapping: staged changes.
4. Dependencies: S6.
5. Preconditions: gitleaks available; configuration present.
6. Action / side effects: scan the staged diff (the pre-commit hook runs the same scan; the pre-push hook scans the outgoing commits).
7. Autonomy: A2.
8. Timeout / retry: 2 minutes / 1 retry on tool error.
9. Idempotency: read-only.
10. Postconditions: `scan-result@0.1.0`.
11. Verification: the scanner exit status and the report agree; findings list rule, file and line only.
12. Failure routes: any finding → unstage, do not commit or push, `blocked`, `secret_detected` alert; scanner missing or erroring → `blocked` (never push unscanned).
13. Escalation triggers: every finding (owner decides: remove the secret and rotate it, or approve an allowlist entry for a proven false positive).
14. Evidence: scan result (no secret values).

**S8 — Commit and push**
1. Objective: a reviewable history in the private repository.
2. Capability: `nova.mirror.config.commit_push`.
3. Input mapping: clean staged diff; deploy key reference.
4. Dependencies: S7 clean.
5. Preconditions: remote reachable; fast-forward possible.
6. Action / side effects: commit only if something changed; push fast-forward with the repository-scoped deploy key; hooks run again; remote push protection is an extra layer.
7. Autonomy: A2.
8. Timeout / retry: 60 s / 2 retries on transport errors only.
9. Idempotency: no change → no commit; pushing the same commit twice is a no-op.
10. Postconditions: remote ref equals the local commit.
11. Verification: read the remote ref.
12. Failure routes: rejected (non-fast-forward or push protection) → `blocked`, alert; never force-push.
13. Escalation triggers: mirror not pushed for more than `stale_after_hours` → `mirror_stale` alert.
14. Evidence: commit ID.

**S9 — Weekly vulnerability scan**
1. Objective: know what known vulnerabilities run in production.
2. Capability: `nova.security.vuln_scan.scan_images`.
3. Input mapping: running images and digests (or the configured list).
4. Dependencies: none (own schedule).
5. Preconditions: vulnerability database updated within 48 h.
6. Action / side effects: Trivy per image digest with severity HIGH and CRITICAL; compute fixable counts and new-since-last-week.
7. Autonomy: A2.
8. Timeout / retry: 30 minutes / 1 retry.
9. Idempotency: read-only.
10. Postconditions: `vuln-report@0.1.0`.
11. Verification: every running image scanned; `db_fresh = true`; unpinned images flagged.
12. Failure routes: database stale → `partially_completed`, `scan_db_stale` alert, never "0 findings"; image scan fails → that image `unknown`.
13. Escalation triggers: any HIGH or CRITICAL → one `vuln_found` alert with counts per image and fixable totals; no automatic upgrade.
14. Evidence: report.

**S10 — Monthly restore drill (human-assisted)**
1. Objective: prove an archive can be restored.
2. Capability: `nova.backup.drill.run`.
3. Input mapping: `drill-request@0.1.0` (latest archive; isolated scratch environment).
4. Dependencies: S4 (an archive exists).
5. Preconditions: the operator supplies the private key in the scratch environment only for the drill; the scratch environment has no network path to production and no production credentials.
6. Action / side effects: decrypt in scratch; verify checksums; SQLite integrity; row counts compared with the archive manifest; boot the applications against the restored data with outbound actions disabled; count workflows; destroy the scratch environment and the key copy.
7. Autonomy: A0/A1 — the operator starts and supervises it.
8. Timeout / retry: 2 hours / none.
9. Idempotency: scratch per drill ID.
10. Postconditions: `drill-report@0.1.0`; archive `drilled_at` set on pass.
11. Verification: all selected checks `pass`.
12. Failure routes: any check fails → `drill_failed` alert; the next nightly run is flagged until a drill passes.
13. Escalation triggers: no passing drill in 35 days → `drill_due` alert.
14. Evidence: drill report.

**S11 — Report, alert and update the board**
1. Objective: honest status, quiet when healthy.
2. Capability: `nova.notify.owner.send`; `nova.jobs.registry.report`.
3. Input mapping: run outputs → `alert@0.1.0` and `job-report@0.1.0` rows `backup`, `mirror`, `vuln-scan`, `restore-drill`.
4. Dependencies: the job's last step.
5. Preconditions: none.
6. Action / side effects: send alerts only for failure, staleness, secrets, findings or drills; update board rows with `last_success` separate from the last run; release the lock.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 2 retries.
9. Idempotency: alert key per job, night and kind.
10. Postconditions: alerts delivered; board updated.
11. Verification: delivery receipts; a job with no success within `stale_after_hours` shows `late`, and a board without fresh reports shows `unknown`.
12. Failure routes: channel down → board still shows the state; run `partially_completed`.
13. Escalation triggers: none.
14. Evidence: message IDs, board timestamps.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| Encryption and keys | no plaintext archive in the store; header present; private key absent from the host; keys stored apart from archives | S4 checks, key inventory | holds | delete, `failed`, escalate |
| SQLite consistency | snapshot via online API; `integrity_check` ok | S2/S3 | holds | exclude item |
| Retention safety | newest good and ≥ `keep_min` good archives remain | prune plan | holds | skip pruning |
| Off-site truth | remote checksum equals local, or `not_configured` reported | S4 | holds | alert |
| Mirror scope | changed paths ⊆ whitelist; no deny pattern | S6 diff | holds | exclude, report |
| Secret gate | every push preceded by a clean scan; scanner present | S7/S8 logs | 100 % | block push |
| No secret echo | findings, logs and alerts contain no secret value | log scan | none | rotate the secret, purge logs |
| Credential hygiene | no password or connection string in any process argument or error | argument spy in tests, log scan | none | stop, escalate |
| Scan freshness | vulnerability database fresh; all running images scanned | S9 | holds | report `partially_completed` |
| Drill recency | a passed drill within 35 days | drill reports | holds | `drill_due` alert |
| Output schemas | all run outputs valid | validator | valid | fix; never fake |

A backup job exiting 0 proves only that it ran; only integrity checks, checksums and drills prove the archive is usable.

## J. Self-correction and recovery

- **Failure classes:** preflight failure, snapshot failure, integrity failure, encryption failure, off-site failure, retention conflict, deny-pattern file, secret finding, scanner missing, push rejection, stale vulnerability database, drill failure.
- **Permitted corrections:** retry a snapshot once; cold-stop fallback only for `cold_ok` items in the window; retry uploads after a checksum check; skip pruning when unsafe; exclude deny-pattern files; retry a push on transport errors.
- **Forbidden corrections:** write plaintext; copy live database files; prune below protections; allowlist a finding; force-push; push unscanned; report a stale-database scan as clean; print credentials to debug.
- **Maximum correction cycles:** 1 per step.
- **Reverification required after correction:** yes — rerun that step's verification.
- **Stop conditions:** hard constraint fails; correction budget used.
- **Recovery or compensation procedure:** restart any stopped service; abort partial uploads; delete temp snapshots; rotate any secret that reached a remote.
- **What must be reported:** items excluded, fallbacks used, off-site state, findings (rule and path), image counts, drill results.

## K. Independent challenge

- **Assumptions to challenge:** the archive restores (only a drill proves it); the whitelist still matches reality (new directories with secrets can appear); "0 vulnerabilities" means the scanner had a current database and saw every running image.
- **Alternative explanation:** a much smaller archive than usual may mean an item silently became empty or unreadable, not that data was cleaned up — flagged at S3.
- **Contradictory evidence:** the mirror shows a config change that the activity log never recorded; the board shows backups green while the off-site copy has failed for days.
- **Could mislead:** a green backup job without off-site copies; a clean secret scan of a whitelist that excludes the file containing the secret (which then is never backed up anywhere reviewable).
- **Separate verifier:** the monthly drill runs from the archive alone in a scratch environment, independent of the job that created it; the remote's push protection scans independently of the local hooks.
- **Findings requiring disclosure:** size anomalies, deny-pattern exclusions, unpinned images, missing off-site copies.

## L. Human escalation

- **Triggers:** secret detected; backup failed or stale; off-site failing 3 nights; HIGH/CRITICAL findings; stale vulnerability database; drill failed or due; service unhealthy after cold-stop.
- **Why the agent cannot decide:** rotating secrets, approving allowlists, upgrading images and running restores are owner or operator decisions with security consequences.
- **Information presented:** job, failure class, affected item or image, counts, rule ID and file path (never the secret), last good time.
- **Decision requested:** remove and rotate the secret or approve a false positive; plan an image upgrade; fix storage; schedule a drill.
- **Options and consequences:** leaving a finding unresolved keeps the mirror blocked (configuration history stops); ignoring vulnerabilities keeps the risk.
- **Safe behaviour while waiting:** backups continue; the mirror stays blocked; scans continue weekly.
- **If no response:** the same alert repeats at most weekly; the board stays honest.
- **Resume:** the next scheduled run after the fix; a resolved finding passes the scan and pushes.

## M. Completion conditions

Mandatory: no plaintext archive anywhere; snapshots consistent and verified; retention protections held; every push preceded by a clean scan; no secret or credential echoed or placed in process arguments; vulnerability results only reported as clean with a fresh database; evidence references present; limitations disclosed.

- `completed` — (backup) every item archived, sealed, verified and copied off-site; (mirror) clean scan and push, or nothing changed; (scan) all images scanned with a fresh database; (drill) all checks passed.
- `partially_completed` — some items excluded; off-site not configured or failed; deny-pattern exclusions; stale vulnerability database; alert delivery failed.
- `blocked` — preflight failed (key missing, disk low); secret detected or scanner missing; push rejected.
- `failed` — encryption failed, no item could be archived, or the drill failed.
- `cancelled` — lock held by another run, or the operator stopped the job (temp data cleaned, services running).

A backup without an off-site copy is never `completed`; a mirror blocked by a finding is never `completed`.

## N. Final output contract

Return per job: run ID; recipe ID and version; job; status; one-line outcome ("2/2 items archived, off-site copied, 1 pruned"); `backup_run`, `mirror_run`, `vuln_report` or `drill_report`; step statuses; verification results and failed checks; evidence references (archive IDs and checksums, commit ID, report IDs); approvals and scope (enablement scope, cold-stop window, allowlist approvals); side effects (archives, uploads, pushes, service restarts) and reconciliation status; corrections; unresolved challenges; limitations; escalation state; learning candidate. Never include secret values, keys, connection strings or archive contents.

## O. Learning and revision

- **Outcome metrics:** backup success rate, archive size trend, off-site success rate, time since last passed drill, mirror block rate and finding types, HIGH/CRITICAL counts per image over time, time to remediate.
- **Evidence retained:** run records, checksums, scan and drill reports, per retention policy.
- **Data minimisation:** no contents, secret values or credentials in metrics or reports.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}`.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review; changes to encryption, the secret gate, the whitelist or credential handling need security review and cannot be applied by an automated repair.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (test host, fictional data) | archives sealed, verified, copied; clean mirror push; scan report | UNVERIFIED on Skill Port; encrypted backups, gitleaks-gated mirror and weekly scan observed in the reference implementation |
| WAL consistency (writes during the snapshot) | snapshot passes `integrity_check` | TBD |
| Missing input (no items) | `blocked` | TBD |
| Invalid input (`keep_min: 1`) | raised to 2, disclosed | TBD |
| Missing capability (encryption tool or recipient key absent) | `blocked`; nothing written, never plaintext | TBD |
| Disk nearly full | `blocked` before snapshot | TBD |
| Permission denied (deploy key read-only) | push rejected, `blocked`, alert | TBD |
| Unauthorized scope (file outside the whitelist, or a deny-pattern file inside it) | not copied; deny-pattern files reported | TBD |
| Planted test secret in a whitelisted file | push blocked; alert shows rule and path only | TBD |
| Scanner missing | `blocked`; nothing pushed | TBD |
| Tenant isolation | separate backup stores and repositories per tenant | TBD |
| Timeout on upload | remote checksum checked before retry | TBD |
| Duplicate / replay (job started twice) | lock lets one run; second `cancelled` | TBD |
| False success (job exits 0 but archive corrupted) | checksum or drill fails; not counted good | TBD |
| Failed verification (snapshot integrity fails) | item excluded, `partially_completed` | TBD |
| Retention protection | newest good and `keep_min` never deleted | TBD |
| Vulnerability scan (HIGH finding in a test image; stale database) | one `vuln_found` alert, no upgrade; stale database → `partially_completed`, never "clean" | TBD |
| Credential hygiene (argument spy and forced timeout) | no password in arguments, logs or errors | UNVERIFIED on Skill Port; pattern observed in the reference implementation |
| Restore drill | scratch restore passes all checks | UNVERIFIED on Skill Port; restore dry run observed in the reference implementation |
| Correction budget exhausted | stops safely | TBD |
| Escalation (drill overdue) | `drill_due` alert | TBD |
| Cancellation mid-backup | temp data deleted, services running, no partial archive marked good | TBD |
| Compensation (cold-stop copy fails) | service restarted and healthy | TBD |
| Output schema and status | valid outputs; correct status | TBD |
| Compatibility / regression (new SQLite or Trivy version) | snapshot and scan still pass | TBD |

All tests run on a test host with fictional data and a throwaway repository; never against production data or the real mirror. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs and whether the platform runs host-level jobs at all — TBD (owner: publisher, with Skill Port).
- Off-site provider and its cost — TBD (owner: operator); strongly recommended but optional in 0.1.0.
- Managed cloud databases are out of scope (provider point-in-time recovery is assumed and should be confirmed by the operator).
- gitleaks finds known secret shapes; it can miss novel formats, which is why the whitelist and deny list remain the first line of defence.
- The vulnerability baseline of third-party base images may stay high until deliberate upgrades; the recipe reports, it does not patch.
- Private-key custody for the drill is a human process; losing the off-host private key makes every archive unrecoverable.
