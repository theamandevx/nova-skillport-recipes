---
name: nova-subscription-audit
description: "Skill Port recipe: Subscription audit. Once a month, the agent scans the user's transaction history, payments log and payment schedule for recurring charges."
---

# Subscription audit

Once a month, the agent scans the user's transaction history, payments log and payment schedule for recurring charges. It works out each subscription's cadence and monthly-equivalent cost, totals them per currency and in one reporting currency, and flags duplicates, price increases, trials that turned paid, and charges after a cancellation. It sends a short digest with suggestions. It is strictly read-only: it never cancels anything and never changes a record.

This folder is a **Skill Port recipe** written to Skill Port Recipe Standard v1. It is a **Draft (0.1.0)**: ready for review, not approved for execution. No capability in it is registered with Skill Port, and every binding is marked `UNVERIFIED`.

- **Recipe ID:** `nova.finance.subscription-audit` (proposed until assigned by Skill Port)
- **Version:** 0.1.0 · **Status:** Draft
- **Category:** finance
- **Tags:** subscriptions, recurring-charges, audit, monthly-digest, duplicates, price-increase, read-only
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
