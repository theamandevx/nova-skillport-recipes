---
name: nova-net-worth-overview
description: "Skill Port recipe: Net worth overview. On request, the agent reads every account balance from the finance hub."
---

# Net worth overview

On request, the agent reads every account balance from the finance hub. The hub mirrors the user's external personal-finance app over REST, and that app is the source of truth. The agent converts each native balance to one reporting currency with the latest verified exchange rates and returns assets, amounts owed and net worth, with per-currency subtotals. It is strictly read-only, and it flags anything stale: cached balances, a provider sync in progress, old balances, or stale or unverified rates.

This folder is a **Skill Port recipe** written to Skill Port Recipe Standard v1. It is a **Draft (0.1.0)**: ready for review, not approved for execution. No capability in it is registered with Skill Port, and every binding is marked `UNVERIFIED`.

- **Recipe ID:** `nova.finance.net-worth-overview` (proposed until assigned by Skill Port)
- **Version:** 0.1.0 · **Status:** Draft
- **Category:** finance
- **Tags:** net-worth, balances, multi-currency, fx-conversion, read-only, stale-data
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
