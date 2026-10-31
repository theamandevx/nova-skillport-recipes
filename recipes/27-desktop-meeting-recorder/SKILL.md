---
name: nova-desktop-meeting-recorder
description: "Skill Port recipe: Desktop meeting recorder. Only after the user explicitly presses Record in a paired desktop app, the app captures the user's microphone and the computer's meeting audio on two separate tracks behind a visible recording indicator, stops cleanly on Stop or when the computer sleeps, relays audio chunks transiently for transcription, and turns the transcript into structured, grounded minutes."
---

# Desktop meeting recorder

Only after the user explicitly presses Record in a paired desktop app, the app captures the user's microphone and the computer's meeting audio on two separate tracks behind a visible recording indicator, stops cleanly on Stop or when the computer sleeps, relays audio chunks transiently for transcription, and turns the transcript into structured, grounded minutes. It never starts recording on its own.

This folder is a **Skill Port recipe** written to Skill Port Recipe Standard v1. It is a **Draft (0.1.0)**: ready for review, not approved for execution. No capability in it is registered with Skill Port, and every binding is marked `UNVERIFIED`.

- **Recipe ID:** `nova.calendar.desktop-meeting-recorder` (proposed until assigned by Skill Port)
- **Version:** 0.1.0 · **Status:** Draft
- **Category:** productivity
- **Tags:** meetings, desktop-capture, transcription, minutes, consent, no-silent-capture, sleep-safe
- **Vedaxi task type:** none (not one of the seven task types Vedaxi supports today)
- **Evidence status:** Unverified

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
