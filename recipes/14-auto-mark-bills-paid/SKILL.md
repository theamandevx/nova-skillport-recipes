---
name: nova-auto-mark-bills-paid
description: "Skill Port recipe: Auto-mark bills paid. When a receipt, tax invoice or 'payment received / renewed' message is detected (or the user says a bill is paid), the agent records it in a payments log."
---

# Auto-mark bills paid

When a receipt, tax invoice or "payment received / renewed" message is detected (or the user says a bill is paid), the agent records it in a payments log. It then reconciles the payment against open obligations: by invoice or order number first, otherwise by product, amount, currency and timing. A matched one-off bill is marked paid; a matched recurring bill moves to its next cycle. This is alert and paid-tracking only: it never changes account balances and never logs transactions.

This folder is a **Skill Port recipe** written to Skill Port Recipe Standard v1. It is a **Draft (0.1.0)**: ready for review, not approved for execution. No capability in it is registered with Skill Port, and every binding is marked `UNVERIFIED`.

- **Recipe ID:** `nova.finance.auto-mark-bills-paid` (proposed until assigned by Skill Port)
- **Version:** 0.1.0 · **Status:** Draft
- **Category:** finance
- **Tags:** payments, reconciliation, receipts, invoice-matching, payments-log, paid-tracking
- **Vedaxi task type:** none (no supported task type covers this today)
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
