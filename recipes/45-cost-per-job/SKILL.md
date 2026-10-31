---
name: nova-cost-per-job
description: "Skill Port recipe: Cost per job. Usage collectors read AI calls from the automation hub's execution records and the assistant's own session store, attribute tokens and spend to the job that caused them (`meta.job`), and append them to a usage ledger going forward only, with high-water marks, replay guards and honest cost bases."
---

# Cost per job

Usage collectors read AI calls from the automation hub's execution records and the assistant's own session store, attribute tokens and spend to the job that caused them (`meta.job`), and append them to a usage ledger going forward only, with high-water marks, replay guards and honest cost bases. A daily check alerts the owner when spend exceeds about 2.5× the 7-day average.

This folder is a **Skill Port recipe** written to Skill Port Recipe Standard v1. It is a **Draft (0.1.0)**: ready for review, not approved for execution. No capability in it is registered with Skill Port, and every binding is marked `UNVERIFIED`.

- **Recipe ID:** `nova.ops.cost-per-job` (proposed until assigned by Skill Port)
- **Version:** 0.1.0 · **Status:** Draft
- **Category:** operations
- **Tags:** cost-tracking, ai-usage, tokens, attribution, high-water-mark, replay-guard, spend-alert
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
