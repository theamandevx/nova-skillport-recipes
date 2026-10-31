---
name: nova-backup-secret-scan-security-scan
description: "Skill Port recipe: Backups, secret scan and security scan. Nightly, the agent takes consistent, encrypted backups of the assistant's state and the automation hub, applies retention and (recommended) copies them off-site, and mirrors whitelisted configuration one way into a private git repository behind a secret scanner that blocks any push containing a secret."
---

# Backups, secret scan and security scan

Nightly, the agent takes consistent, encrypted backups of the assistant's state and the automation hub, applies retention and (recommended) copies them off-site, and mirrors whitelisted configuration one way into a private git repository behind a secret scanner that blocks any push containing a secret. Weekly, it scans running container images for vulnerabilities and alerts on any HIGH or CRITICAL finding. A monthly restore drill proves the backups work, and database passwords never appear on a command line.

This folder is a **Skill Port recipe** written to Skill Port Recipe Standard v1. It is a **Draft (0.1.0)**: ready for review, not approved for execution. No capability in it is registered with Skill Port, and every binding is marked `UNVERIFIED`.

- **Recipe ID:** `nova.ops.backup-secret-scan-security-scan` (proposed until assigned by Skill Port)
- **Version:** 0.1.0 · **Status:** Draft
- **Category:** security
- **Tags:** encrypted-backups, sqlite-wal, restore-drill, retention, config-mirror, gitleaks, trivy, credential-hygiene
- **Vedaxi task type:** none supported today
- **Evidence status:** Partially verified

## Files

| File | Purpose |
|---|---|
| `RECIPE.md` | The full recipe: sections A–Q of the canonical template. Follow it exactly, including every approval checkpoint, stop condition and completion rule. |
| `manifest.draft.yaml` | The capability register (Standard §2.6 shape), marked DRAFT pending Skill Port's approved schema. |
| `VALIDATION.md` | The validation report (§7) and the §8 review checklist, with honest status. |

## How an agent uses this recipe

1. Read `RECIPE.md` Sections C–G before acting: applicability, inputs, constraints, CRUD scope and autonomy/approval policy.
2. Run the steps in Section H in order, only through capabilities the runtime has registered and authorized. If a required capability is missing, stop as Section H and J describe.
3. Verify after each step (Section I), and report one of the Section M outcomes using the Section N output contract.

This recipe declares requirements only. It grants no permissions; the runtime enforces authentication, consent, authorization and tenant scope.
