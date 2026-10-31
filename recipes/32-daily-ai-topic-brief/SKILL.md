---
name: nova-daily-ai-topic-brief
description: "Skill Port recipe: Daily AI topic brief. Every morning at the owner's chosen time, the agent takes the freshest trending items for the owner's subject (AI by default), picks the top five, writes one sourced line for each and delivers the brief to the owner's chat."
---

# Daily AI topic brief

Every morning at the owner's chosen time, the agent takes the freshest trending items for the owner's subject (AI by default), picks the top five, writes one sourced line for each and delivers the brief to the owner's chat. The scheduled job runs with a declared fallback model on a **different provider**, and any failure is reported to the owner instead of passing silently.

This folder is a **Skill Port recipe** written to Skill Port Recipe Standard v1. It is a **Draft (0.1.0)**: ready for review, not approved for execution. No capability in it is registered with Skill Port, and every binding is marked `UNVERIFIED`.

- **Recipe ID:** `nova.knowledge.daily-ai-topic-brief` (proposed until assigned by Skill Port)
- **Version:** 0.1.0 · **Status:** Draft
- **Category:** research
- **Tags:** daily-brief, scheduled, trending-topics, sources, owner-chat, model-fallback, failure-alerts, reliability
- **Vedaxi task type:** none of the seven supported today maps directly
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
