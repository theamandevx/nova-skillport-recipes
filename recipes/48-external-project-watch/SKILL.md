---
name: nova-external-project-watch
description: "Skill Port recipe: External project watch. Another system the user runs (an 'external project') pushes a read-only status rollup on a schedule to an authenticated ingest webhook."
---

# External project watch

Another system the user runs (an "external project") pushes a read-only status rollup on a schedule to an authenticated ingest webhook. The agent stores each snapshot, shows the latest on a dashboard with its age, sends the owner red alerts when the numbers turn bad, and raises one alert when the feed goes stale. Nova stores no credential that could reach into the external system.

This folder is a **Skill Port recipe** written to Skill Port Recipe Standard v1. It is a **Draft (0.1.0)**: ready for review, not approved for execution. No capability in it is registered with Skill Port, and every binding is marked `UNVERIFIED`.

- **Recipe ID:** `nova.ops.external-project-watch` (proposed until assigned by Skill Port)
- **Version:** 0.1.0 · **Status:** Draft
- **Category:** operations
- **Tags:** monitoring, push-model, webhook-ingest, hmac, dashboard, red-alerts, stale-feed-detection
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
