---
name: nova-voiceover
description: "Skill Port recipe: Voiceover. From an approved script, the agent checks the text-to-speech provider's readiness and remaining character quota, confirms the voice and the estimated cost, generates the audio, stores the file and records characters and cost exactly once."
---

# Voiceover

From an approved script, the agent checks the text-to-speech provider's readiness and remaining character quota, confirms the voice and the estimated cost, generates the audio, stores the file and records characters and cost exactly once. A cloned voice is never created or used without a recorded, unrevoked consent from the voice's owner.

This folder is a **Skill Port recipe** written to Skill Port Recipe Standard v1. It is a **Draft (0.1.0)**: ready for review, not approved for execution. No capability in it is registered with Skill Port, and every binding is marked `UNVERIFIED`.

- **Recipe ID:** `nova.content.voiceover` (proposed until assigned by Skill Port)
- **Version:** 0.1.0 · **Status:** Draft
- **Category:** design
- **Tags:** text-to-speech, voiceover, audio, voice-selection, quota-check, cost-tracking, voice-cloning-consent
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
