---
name: nova-find-research-records
description: "Skill Port recipe: Find research records. On a research or current-facts question, the agent searches through a self-hosted metasearch engine (or a scholarly-record index), fetches and reads the real sources, writes a brief in which every claim carries an inline citation to a source read in this run, and saves the brief to the user's research library."
---

# Find research records

On a research or current-facts question, the agent searches through a self-hosted metasearch engine (or a scholarly-record index), fetches and reads the real sources, writes a brief in which every claim carries an inline citation to a source read in this run, and saves the brief to the user's research library. It never answers a current-facts question from model memory.

This folder is a **Skill Port recipe** written to Skill Port Recipe Standard v1. It is a **Draft (0.1.0)**: ready for review, not approved for execution. No capability in it is registered with Skill Port, and every binding is marked `UNVERIFIED`.

- **Recipe ID:** `nova.research.find-research-records` (proposed until assigned by Skill Port)
- **Version:** 0.1.0 · **Status:** Draft
- **Category:** research
- **Tags:** web-research, citations, metasearch, scholarly-records, grounding, research-brief
- **Vedaxi task type:** Find research records
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
