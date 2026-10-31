---
name: nova-ai-quota-monitor
description: "Skill Port recipe: AI quota monitor. Every 30 minutes the agent snapshots each AI subscription's usage window (percent used, window length, reset time) and each pay-as-you-go provider's credit balance, keeps the last good snapshot when a fetch fails, shows any snapshot older than 3 hours as `stale` rather than '0 % used', and alerts the owner once when a window crosses 75 % (warning) or 90 % (critical) or a balance runs low or dry."
---

# AI quota monitor

Every 30 minutes the agent snapshots each AI subscription's usage window (percent used, window length, reset time) and each pay-as-you-go provider's credit balance, keeps the last good snapshot when a fetch fails, shows any snapshot older than 3 hours as `stale` rather than "0 % used", and alerts the owner once when a window crosses 75 % (warning) or 90 % (critical) or a balance runs low or dry.

This folder is a **Skill Port recipe** written to Skill Port Recipe Standard v1. It is a **Draft (0.1.0)**: ready for review, not approved for execution. No capability in it is registered with Skill Port, and every binding is marked `UNVERIFIED`.

- **Recipe ID:** `nova.ops.ai-quota-monitor` (proposed until assigned by Skill Port)
- **Version:** 0.1.0 · **Status:** Draft
- **Category:** operations
- **Tags:** ai-quota, usage-window, credit-balance, stale-detection, threshold-alerts, fallback-check
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
