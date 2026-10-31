# Nova recipes for Skill Port

52 recipes written to **Skill Port Recipe Standard v1** (`SKILLPORT_RECIPE_STANDARD-1.md`, included here). Each one turns a working Nova feature into a reviewable package: capability register, inputs and outputs with schemas, a step-by-step execution plan with approval points, verification, escalation, completion states and tests.

Every recipe is a **Draft (0.1.0)**. Nothing here is registered with Skill Port or tested on Vedaxi yet; every binding is marked `UNVERIFIED` and every recipe's `VALIDATION.md` lists what is still missing.

## What is in this package

| Path | What it is |
|---|---|
| `recipes/<NN>-<name>/SKILL.md` | The package manifest Skill Port's importer requires (front matter `name`, `description`) |
| `recipes/<NN>-<name>/RECIPE.md` | The recipe: sections A–Q of the standard's canonical template |
| `recipes/<NN>-<name>/manifest.draft.yaml` | The capability register in the standard's manifest template (§2.6), marked DRAFT |
| `recipes/<NN>-<name>/VALIDATION.md` | The validation report (§7) with the §8 review checklist |
| `SUBMISSION.md` | How to get these recipes onto Skill Port |
| `CONVENTIONS.md` | The shared rules every recipe follows (IDs, bindings, schemas, safety, privacy) |
| `SKILLPORT_RECIPE_STANDARD-1.md` | The standard itself |

## Submit first: the 7 that match a Vedaxi task type today

| # | Recipe | Vedaxi task type | Reference implementation |
|---|---|---|---|
| 01 | [Find research records](recipes/01-find-research-records/RECIPE.md) | Find research records | Built |
| 02 | [Remember a note](recipes/02-remember-a-note/RECIPE.md) | Remember a note | Built |
| 03 | [Find a saved memory](recipes/03-find-a-saved-memory/RECIPE.md) | Find a saved memory | Built |
| 04 | [Save a note draft](recipes/04-save-a-note-draft/RECIPE.md) | Save a note draft | Built |
| 05 | [Prepare a call](recipes/05-prepare-a-call/RECIPE.md) | Prepare a call | Built |
| 06 | [Prepare an SMS](recipes/06-prepare-an-sms/RECIPE.md) | Prepare an SMS | Partly built |
| 07 | [Prepare a WhatsApp message](recipes/07-prepare-a-whatsapp-message/RECIPE.md) | Prepare a WhatsApp message | Built |

The other recipes are complete to the standard but need a task type that Vedaxi does not offer yet. They can be submitted as Skill Port recipes now and run once the platform supports them.

## Full catalogue

**Reference implementation** is how far the feature got in Nova: *Built* (working), *Partly built*, *Built, blocked* (works but waits on an external account or test) or *Removed* (taken out, rebuild planned). **Evidence** is the recipe's own Section A rating. No recipe claims evidence on Skill Port or Vedaxi.

### Research & knowledge

| # | Recipe | Recipe ID | Reference implementation | Evidence |
|---|---|---|---|---|
| 01 | [Find research records](recipes/01-find-research-records/RECIPE.md) | `nova.research.find-research-records` | Built | Partially verified |
| 29 | [Document Q&A](recipes/29-document-qa/RECIPE.md) | `nova.knowledge.document-qa` | Built | Partially verified |
| 30 | [Trending topics radar](recipes/30-trending-topics-radar/RECIPE.md) | `nova.knowledge.trending-topics-radar` | Built | Partially verified |
| 31 | [Competitor news monitor](recipes/31-competitor-news-monitor/RECIPE.md) | `nova.knowledge.competitor-news-monitor` | Built | Partially verified |
| 32 | [Daily AI topic brief](recipes/32-daily-ai-topic-brief/RECIPE.md) | `nova.knowledge.daily-ai-topic-brief` | Built | Partially verified |

### Notes & memory

| # | Recipe | Recipe ID | Reference implementation | Evidence |
|---|---|---|---|---|
| 02 | [Remember a note](recipes/02-remember-a-note/RECIPE.md) | `nova.memory.remember-a-note` | Built | Partially verified |
| 03 | [Find a saved memory](recipes/03-find-a-saved-memory/RECIPE.md) | `nova.memory.find-a-saved-memory` | Built | Partially verified |
| 04 | [Save a note draft](recipes/04-save-a-note-draft/RECIPE.md) | `nova.memory.save-a-note-draft` | Built | Partially verified |

### Calls & messages

| # | Recipe | Recipe ID | Reference implementation | Evidence |
|---|---|---|---|---|
| 05 | [Prepare a call](recipes/05-prepare-a-call/RECIPE.md) | `nova.comms.prepare-a-call` | Built | Partially verified |
| 06 | [Prepare an SMS](recipes/06-prepare-an-sms/RECIPE.md) | `nova.comms.prepare-an-sms` | Partly built | Unverified |
| 07 | [Prepare a WhatsApp message](recipes/07-prepare-a-whatsapp-message/RECIPE.md) | `nova.comms.prepare-a-whatsapp-message` | Built | Partially verified |

### Email & inbox

| # | Recipe | Recipe ID | Reference implementation | Evidence |
|---|---|---|---|---|
| 08 | [Inbox triage](recipes/08-inbox-triage/RECIPE.md) | `nova.inbox.inbox-triage` | Built | Partially verified |
| 09 | [Daily "needs you" digest](recipes/09-daily-needs-you-digest/RECIPE.md) | `nova.inbox.daily-needs-you-digest` | Built | Partially verified |
| 10 | [Draft a reply](recipes/10-draft-a-reply/RECIPE.md) | `nova.inbox.draft-a-reply` | Built | Partially verified |
| 11 | [Dismiss an attention item](recipes/11-dismiss-an-attention-item/RECIPE.md) | `nova.inbox.dismiss-an-attention-item` | Built | Partially verified |

### Money

| # | Recipe | Recipe ID | Reference implementation | Evidence |
|---|---|---|---|---|
| 12 | [Upcoming payments radar](recipes/12-upcoming-payments-radar/RECIPE.md) | `nova.finance.upcoming-payments-radar` | Built | Partially verified |
| 13 | [Payment reminders](recipes/13-payment-reminders/RECIPE.md) | `nova.finance.payment-reminders` | Built | Partially verified |
| 14 | [Auto-mark bills paid](recipes/14-auto-mark-bills-paid/RECIPE.md) | `nova.finance.auto-mark-bills-paid` | Built | Partially verified |
| 15 | [Log transactions from email](recipes/15-log-transactions-from-email/RECIPE.md) | `nova.finance.log-transactions-from-email` | Built | Partially verified |
| 16 | [Log transactions from SMS](recipes/16-log-transactions-from-sms/RECIPE.md) | `nova.finance.log-transactions-from-sms` | Built | Partially verified |
| 17 | [Subscription audit](recipes/17-subscription-audit/RECIPE.md) | `nova.finance.subscription-audit` | Built | Partially verified |
| 18 | [Daily exchange rates](recipes/18-daily-exchange-rates/RECIPE.md) | `nova.finance.daily-exchange-rates` | Built | Partially verified |
| 19 | [Lent and borrowed ledger](recipes/19-lent-borrowed-ledger/RECIPE.md) | `nova.finance.lent-borrowed-ledger` | Built | Partially verified |
| 20 | [Net worth overview](recipes/20-net-worth-overview/RECIPE.md) | `nova.finance.net-worth-overview` | Built | Partially verified |

### Calendar & meetings

| # | Recipe | Recipe ID | Reference implementation | Evidence |
|---|---|---|---|---|
| 21 | [Find free slots](recipes/21-find-free-slots/RECIPE.md) | `nova.calendar.find-free-slots` | Built | Partially verified |
| 22 | [Book a meeting](recipes/22-book-a-meeting/RECIPE.md) | `nova.calendar.book-a-meeting` | Built | Partially verified |
| 23 | [Reschedule, cancel or block time](recipes/23-reschedule-cancel-block-time/RECIPE.md) | `nova.calendar.reschedule-cancel-block-time` | Built | Partially verified |
| 24 | [Meeting prep brief](recipes/24-meeting-prep-brief/RECIPE.md) | `nova.calendar.meeting-prep-brief` | Built | Partially verified |
| 25 | [Meeting minutes](recipes/25-meeting-minutes/RECIPE.md) | `nova.calendar.meeting-minutes` | Built | Partially verified |
| 26 | [Meeting notetaker](recipes/26-meeting-notetaker/RECIPE.md) | `nova.calendar.meeting-notetaker` | Built, blocked | Unverified |
| 27 | [Desktop meeting recorder](recipes/27-desktop-meeting-recorder/RECIPE.md) | `nova.calendar.desktop-meeting-recorder` | Built, blocked | Unverified |
| 28 | [Action item follow-ups](recipes/28-action-item-follow-ups/RECIPE.md) | `nova.calendar.action-item-follow-ups` | Built | Partially verified |

### Content & media

| # | Recipe | Recipe ID | Reference implementation | Evidence |
|---|---|---|---|---|
| 33 | [LinkedIn post in your voice](recipes/33-linkedin-post-in-your-voice/RECIPE.md) | `nova.content.linkedin-post-in-your-voice` | Built, blocked | Partially verified |
| 34 | [Announcement post](recipes/34-announcement-post/RECIPE.md) | `nova.content.announcement-post` | Built | Partially verified |
| 35 | [One source, many posts](recipes/35-one-source-many-posts/RECIPE.md) | `nova.content.one-source-many-posts` | Built | Partially verified |
| 36 | [Post image](recipes/36-post-image/RECIPE.md) | `nova.content.post-image` | Built | Partially verified |
| 37 | [Voiceover](recipes/37-voiceover/RECIPE.md) | `nova.content.voiceover` | Built | Partially verified |
| 38 | [Avatar video](recipes/38-avatar-video/RECIPE.md) | `nova.content.avatar-video` | Built | Partially verified |
| 39 | [Publish to X](recipes/39-publish-to-x/RECIPE.md) | `nova.content.publish-to-x` | Built | Partially verified |

### Sales

| # | Recipe | Recipe ID | Reference implementation | Evidence |
|---|---|---|---|---|
| 40 | [Booking to lead, with a welcome email](recipes/40-booking-to-lead-welcome-email/RECIPE.md) | `nova.sales.booking-to-lead-welcome-email` | Built | Partially verified |
| 41 | [Lead sourcing](recipes/41-lead-sourcing/RECIPE.md) | `nova.sales.lead-sourcing` | Built | Partially verified |
| 42 | [Deal revival nudges](recipes/42-deal-revival-nudges/RECIPE.md) | `nova.sales.deal-revival-nudges` | Built | Partially verified |
| 43 | [Partner outreach drafts](recipes/43-partner-outreach-drafts/RECIPE.md) | `nova.sales.partner-outreach-drafts` | Removed (rebuild planned) | Unverified |

### Operations & monitoring

| # | Recipe | Recipe ID | Reference implementation | Evidence |
|---|---|---|---|---|
| 44 | [Job health monitor](recipes/44-job-health-monitor/RECIPE.md) | `nova.ops.job-health-monitor` | Built | Partially verified |
| 45 | [Cost per job](recipes/45-cost-per-job/RECIPE.md) | `nova.ops.cost-per-job` | Built | Partially verified |
| 46 | [AI quota monitor](recipes/46-ai-quota-monitor/RECIPE.md) | `nova.ops.ai-quota-monitor` | Built | Partially verified |
| 47 | [Device heartbeat](recipes/47-device-heartbeat/RECIPE.md) | `nova.ops.device-heartbeat` | Built | Partially verified |
| 48 | [External project watch](recipes/48-external-project-watch/RECIPE.md) | `nova.ops.external-project-watch` | Built | Partially verified |
| 49 | [Executive brief and business pulse](recipes/49-executive-brief-business-pulse/RECIPE.md) | `nova.ops.executive-brief-business-pulse` | Built | Partially verified |
| 50 | [Approval queue](recipes/50-approval-queue/RECIPE.md) | `nova.ops.approval-queue` | Built | Partially verified |
| 51 | [Activity log](recipes/51-activity-log/RECIPE.md) | `nova.ops.activity-log` | Built | Partially verified |
| 52 | [Backups, secret scan and security scan](recipes/52-backup-secret-scan-security-scan/RECIPE.md) | `nova.ops.backup-secret-scan-security-scan` | Built | Partially verified |

## Privacy

All examples use fictional data (`example.com` addresses, `+1-555-01xx` numbers, names like Jane Doe). No file contains credentials, real people, real businesses or details of a real deployment.
