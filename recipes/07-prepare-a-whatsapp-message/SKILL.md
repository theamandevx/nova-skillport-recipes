---
name: nova-prepare-a-whatsapp-message
description: "Skill Port recipe: Prepare a WhatsApp message. On an explicit request, the agent resolves the recipient, drafts one WhatsApp message, checks the channel (linked-device session health, recipient allowlist, or for business messaging the official API's opt-in, 24-hour window and template rules), and asks the user to approve the exact text."
---

# Prepare a WhatsApp message

On an explicit request, the agent resolves the recipient, drafts one WhatsApp message, checks the channel (linked-device session health, recipient allowlist, or for business messaging the official API's opt-in, 24-hour window and template rules), and asks the user to approve the exact text. It sends only after a yes — through the user's paired linked-device session for personal messages, the WhatsApp Business Platform for customer-facing messages, or by handing the text to the user.

This folder is a **Skill Port recipe** written to Skill Port Recipe Standard v1. It is a **Draft (0.1.0)**: ready for review, not approved for execution. No capability in it is registered with Skill Port, and every binding is marked `UNVERIFIED`.

- **Recipe ID:** `nova.comms.prepare-a-whatsapp-message` (proposed until assigned by Skill Port)
- **Version:** 0.1.0 · **Status:** Draft
- **Category:** productivity
- **Tags:** whatsapp, messaging, drafting, linked-device, business-api, templates, human-approval
- **Vedaxi task type:** Prepare a WhatsApp message
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
