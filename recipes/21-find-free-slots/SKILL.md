---
name: nova-find-free-slots
description: "Skill Port recipe: Find free slots. Given a meeting length and a date range, the agent computes bookable slots from one master busy-store (bookings, manual blocks, busy time synced from connected calendars and busy time parsed from meeting invites), honouring working hours, notice, horizon, buffers and daily limits, and returns them in the requester's time zone with UTC and a source-freshness disclosure."
---

# Find free slots

Given a meeting length and a date range, the agent computes bookable slots from one master busy-store (bookings, manual blocks, busy time synced from connected calendars and busy time parsed from meeting invites), honouring working hours, notice, horizon, buffers and daily limits, and returns them in the requester's time zone with UTC and a source-freshness disclosure. The same operation is exposed to external booking surfaces as an MCP tool.

This folder is a **Skill Port recipe** written to Skill Port Recipe Standard v1. It is a **Draft (0.1.0)**: ready for review, not approved for execution. No capability in it is registered with Skill Port, and every binding is marked `UNVERIFIED`.

- **Recipe ID:** `nova.calendar.find-free-slots` (proposed until assigned by Skill Port)
- **Version:** 0.1.0 · **Status:** Draft
- **Category:** productivity
- **Tags:** calendar, availability, scheduling, slot-engine, busy-store, time-zones, MCP, read-only
- **Vedaxi task type:** none of the seven supported today
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
