---
name: nova-log-transactions-from-sms
description: "Skill Port recipe: Log transactions from SMS. An SMS-forwarder app on the user's phone posts bank and card text messages to an authenticated webhook."
---

# Log transactions from SMS

An SMS-forwarder app on the user's phone posts bank and card text messages to an authenticated webhook. The agent normalises the payload, drops anything that is not a money message (one-time codes are never sent to an LLM), and computes a hashed idempotency key. It then extracts the transaction with an LLM and routes it exactly like the email logger: high-confidence matches inside the user's standing approval are logged, everything else goes to the review queue, and duplicates are reconciled.

This folder is a **Skill Port recipe** written to Skill Port Recipe Standard v1. It is a **Draft (0.1.0)**: ready for review, not approved for execution. No capability in it is registered with Skill Port, and every binding is marked `UNVERIFIED`.

- **Recipe ID:** `nova.finance.log-transactions-from-sms` (proposed until assigned by Skill Port)
- **Version:** 0.1.0 · **Status:** Draft
- **Category:** finance
- **Tags:** transactions, sms, webhook, bank-alerts, idempotency, finance-hub, review-queue
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
