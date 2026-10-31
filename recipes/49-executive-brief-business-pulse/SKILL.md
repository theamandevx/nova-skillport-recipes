---
name: nova-executive-brief-business-pulse
description: "Skill Port recipe: Executive brief and business pulse. Every morning the agent assembles a short executive brief from the user's own data sources (today's meetings, mail that needs a reply, payments coming due, pipeline changes, overnight assistant activity, external project status)."
---

# Executive brief and business pulse

Every morning the agent assembles a short executive brief from the user's own data sources (today's meetings, mail that needs a reply, payments coming due, pipeline changes, overnight assistant activity, external project status). Every week it sends a business pulse with week-over-week changes. Every line carries its source and an as-of timestamp, numbers are copied, never computed by a language model, and nothing is sent when nothing material happened.

This folder is a **Skill Port recipe** written to Skill Port Recipe Standard v1. It is a **Draft (0.1.0)**: ready for review, not approved for execution. No capability in it is registered with Skill Port, and every binding is marked `UNVERIFIED`.

- **Recipe ID:** `nova.ops.executive-brief-business-pulse` (proposed until assigned by Skill Port)
- **Version:** 0.1.0 · **Status:** Draft
- **Category:** operations
- **Tags:** daily-brief, weekly-pulse, digest, cross-source, provenance, deterministic-numbers, quiet-by-default
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
