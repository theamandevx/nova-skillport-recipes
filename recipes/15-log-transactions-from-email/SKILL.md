---
name: nova-log-transactions-from-email
description: "Skill Port recipe: Log transactions from email. On a schedule, the agent reads transaction emails (bank debit/credit alerts, card spends, settlements, refunds) and extracts each one with an LLM."
---

# Log transactions from email

On a schedule, the agent reads transaction emails (bank debit/credit alerts, card spends, settlements, refunds) and extracts each one with an LLM. It logs the transaction to the finance hub only when the account match and the model's confidence are both high, inside a standing scope the user granted. Everything else waits in a review queue for the user's approval. Per-order payment-gateway notices are skipped, and an existing manual entry within ±3 days is detected, so nothing is counted twice.

This folder is a **Skill Port recipe** written to Skill Port Recipe Standard v1. It is a **Draft (0.1.0)**: ready for review, not approved for execution. No capability in it is registered with Skill Port, and every binding is marked `UNVERIFIED`.

- **Recipe ID:** `nova.finance.log-transactions-from-email` (proposed until assigned by Skill Port)
- **Version:** 0.1.0 · **Status:** Draft
- **Category:** finance
- **Tags:** transactions, bank-alerts, card-spend, email-extraction, finance-hub, review-queue, deduplication
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
