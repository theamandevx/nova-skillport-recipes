---
name: nova-inbox-triage
description: "Skill Port recipe: Inbox triage. On a schedule, the agent takes newly synced mail (metadata only; bodies are never stored), skips mail that never needs a human or that another pipeline already owns, and asks an LLM to rate importance, whether a reply is needed and a one-line summary."
---

# Inbox triage

On a schedule, the agent takes newly synced mail (metadata only; bodies are never stored), skips mail that never needs a human or that another pipeline already owns, and asks an LLM to rate importance, whether a reply is needed and a one-line summary. Each result is computed once per RFC Message-ID and copied to every mailbox copy of that message.

This folder is a **Skill Port recipe** written to Skill Port Recipe Standard v1. It is a **Draft (0.1.0)**: ready for review, not approved for execution. No capability in it is registered with Skill Port, and every binding is marked `UNVERIFIED`.

- **Recipe ID:** `nova.inbox.inbox-triage` (proposed until assigned by Skill Port)
- **Version:** 0.1.0 · **Status:** Draft
- **Category:** productivity
- **Tags:** email, triage, importance, needs-reply, one-line-summary, metadata-only, scheduled, privacy
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
