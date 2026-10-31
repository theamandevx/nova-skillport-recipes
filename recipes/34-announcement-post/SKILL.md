---
name: nova-announcement-post
description: "Skill Port recipe: Announcement post. The user shares their own news — a launch, rebrand, milestone or update — as a rough note."
---

# Announcement post

The user shares their own news — a launch, rebrand, milestone or update — as a rough note. The agent turns it into an announcement post that states the news in its first line, uses only facts from the user's brief, and saves the draft to the approval queue. It never invents claims, numbers, customers, partners or quotes.

This folder is a **Skill Port recipe** written to Skill Port Recipe Standard v1. It is a **Draft (0.1.0)**: ready for review, not approved for execution. No capability in it is registered with Skill Port, and every binding is marked `UNVERIFIED`.

- **Recipe ID:** `nova.content.announcement-post` (proposed until assigned by Skill Port)
- **Version:** 0.1.0 · **Status:** Draft
- **Category:** marketing
- **Tags:** announcement, launch, milestone, lead-with-the-news, linkedin, x, writing, approval-queue, no-invented-claims
- **Vedaxi task type:** none of the seven supported today maps directly (closest: Save a note draft)
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
