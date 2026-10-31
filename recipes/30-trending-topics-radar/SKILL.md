---
name: nova-trending-topics-radar
description: "Skill Port recipe: Trending topics radar. On a schedule (and on demand), the agent scans approved public sources for trending items about the user's chosen topics, filters, de-duplicates and scores them, and stores each one as a topic candidate with its source link."
---

# Trending topics radar

On a schedule (and on demand), the agent scans approved public sources for trending items about the user's chosen topics, filters, de-duplicates and scores them, and stores each one as a topic candidate with its source link. The owner can get a short ranked digest, and the items they pick feed content drafting.

This folder is a **Skill Port recipe** written to Skill Port Recipe Standard v1. It is a **Draft (0.1.0)**: ready for review, not approved for execution. No capability in it is registered with Skill Port, and every binding is marked `UNVERIFIED`.

- **Recipe ID:** `nova.knowledge.trending-topics-radar` (proposed until assigned by Skill Port)
- **Version:** 0.1.0 · **Status:** Draft
- **Category:** research
- **Tags:** trends, topic-discovery, news-scan, deduplication, scheduled, content-ideas, sources
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
