---
name: nova-prepare-an-sms
description: "Skill Port recipe: Prepare an SMS. On an explicit request, the agent resolves the recipient, drafts one short SMS, measures its length and encoding, checks opt-out status, quiet hours and the destination country's sender rules, and asks the user to approve the exact text."
---

# Prepare an SMS

On an explicit request, the agent resolves the recipient, drafts one short SMS, measures its length and encoding, checks opt-out status, quiet hours and the destination country's sender rules, and asks the user to approve the exact text. By default the approved text is handed to the user to send from their own phone; sending through an SMS provider is possible only when a provider binding is installed (UNVERIFIED — not built in the reference implementation) and always needs per-message approval.

This folder is a **Skill Port recipe** written to Skill Port Recipe Standard v1. It is a **Draft (0.1.0)**: ready for review, not approved for execution. No capability in it is registered with Skill Port, and every binding is marked `UNVERIFIED`.

- **Recipe ID:** `nova.comms.prepare-an-sms` (proposed until assigned by Skill Port)
- **Version:** 0.1.0 · **Status:** Draft
- **Category:** productivity
- **Tags:** sms, text-message, drafting, opt-out, quiet-hours, sender-id, human-approval
- **Vedaxi task type:** Prepare an SMS
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
