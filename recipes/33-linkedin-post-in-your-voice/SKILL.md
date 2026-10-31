---
name: nova-linkedin-post-in-your-voice
description: "Skill Port recipe: LinkedIn post in your voice. From a topic the user picked (or a short brief), the agent drafts a LinkedIn post in the user's own voice, checks it, and saves it to an approval queue where the user can edit, approve, reject or schedule it."
---

# LinkedIn post in your voice

From a topic the user picked (or a short brief), the agent drafts a LinkedIn post in the user's own voice, checks it, and saves it to an approval queue where the user can edit, approve, reject or schedule it. A post is published only after a per-post approval, and only through the user's own approved LinkedIn app connection. Feature status: **Built, blocked** — drafting works; publishing is blocked until the user's LinkedIn app and OAuth connection are approved and active.

This folder is a **Skill Port recipe** written to Skill Port Recipe Standard v1. It is a **Draft (0.1.0)**: ready for review, not approved for execution. No capability in it is registered with Skill Port, and every binding is marked `UNVERIFIED`.

- **Recipe ID:** `nova.content.linkedin-post-in-your-voice` (proposed until assigned by Skill Port)
- **Version:** 0.1.0 · **Status:** Draft
- **Category:** marketing
- **Tags:** linkedin, writing, personal-brand, voice, approval-queue, human-approval, publishing-blocked
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
