# Nova recipes — authoring conventions (Skill Port Recipe Standard v1)

Every recipe in this package follows `SKILLPORT_RECIPE_STANDARD-1.md` (the standard) exactly. These conventions only fill the gaps the standard leaves to the author. Where they conflict, the standard wins.

## 1. Files per recipe

Each recipe lives in its own folder `recipes/<NN>-<slug>/` (NN = two-digit catalogue number) with exactly four files (`SKILL.md` is the Skill Port package manifest, added in 0.1.1):

1. `RECIPE.md` — the canonical template, sections **A through Q in order, with the exact headings** `## A. Identity and ownership` … `## Q. Open questions and known limitations`. Never omit a section; write `Not applicable — <reason>` where it does not apply.
2. `manifest.draft.yaml` — the capability register using the standard's **"Capability manifest template"** shape from section 2.6 (top-level key `manifest:` with `package:` and `capabilities:`). It must parse as YAML. First line must be the comment `# DRAFT — conceptual authoring shape from Skill Port Recipe Standard v1 §2.6; pending validation against Skill Port's approved JSON Schema.`
3. `VALIDATION.md` — the "short validation report" from §7 of the standard: missing fields, contradictions, unverified dependencies, risk concerns, tests still needed, plus a checklist run of §8 (capability-register review checklist) with honest ticks.

Do **not** produce a JSON file: Skill Port's JSON Schema is not published, so a JSON draft would invent fields (the standard forbids that).

## 2. Identity

- **Recipe ID:** `nova.<area>.<slug>` — always followed by "(proposed until assigned by Skill Port)".
- **Areas:** `research`, `memory`, `comms`, `inbox`, `finance`, `calendar`, `knowledge`, `content`, `sales`, `ops`.
- **Version:** `0.1.0`. **Status:** `Draft`.
- **Owner or publisher:** `Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)`.
- **Category:** one of Skill Port's categories: development, data analysis, marketing, finance, writing, productivity, security, design, operations, research.
- **Evidence status:** `Partially verified` when the behaviour runs in Nova's reference implementation; say plainly that nothing has been verified on Skill Port / Vedaxi. Use `Unverified` for anything only partly built or removed.
- **Changelog:** `0.1.0 — Initial draft converted from the Nova reference implementation.`
- If the recipe maps to a task type Vedaxi supports today, state it in Section A under **Tags** and in Section C: `Vedaxi task type: <exact name>`. The seven supported today are: Find research records · Remember a note · Find a saved memory · Prepare a call · Prepare an SMS · Prepare a WhatsApp message · Save a note draft.

## 3. Capability IDs, bindings and status

- Nothing from Nova is registered in Skill Port's registry. **Every capability, operation, binding and variant is a PROPOSED object.** In Section B, put them under a heading `### Proposed objects (not registry-confirmed)` and add a line `### Registry-confirmed objects` → `None. No object in this recipe is registered with Skill Port yet.`
- IDs: capability `nova.<domain>.<capability>`, operation `nova.<domain>.<capability>.<operation>`, variant `<operation_id>.<variant>`, binding `binding://nova/<domain>/<name>` with `status: UNVERIFIED`.
- Describe the **reference implementation** in neutral, portable terms (for example "a webhook gateway on the automation hub that calls a database dispatcher function", "an MCP tool `get_available_slots`", "a speech-to-text provider"). You may name widely used vendors/technologies (Telegram, WhatsApp Business API, Gmail API, Google Calendar, PostgreSQL, pgvector, n8n, an MCP server, a telephony provider with bidirectional audio streaming, Whisper, SearXNG). Never name the owner's own businesses, clients, partners or projects.
- Health status for every variant: `unverified` (the standard: never label an untested tool healthy). Test refs: `TBD`.
- Schema refs: `schema://nova/<slug>/<name>@0.1.0`, and **define each schema inline in Section D** as a compact JSON Schema (type, required, properties, enums, formats) whose `$id` equals the ref. Every schema the manifest references must be defined this way; a prose sketch does not count. Several schemas may share one ```json block, back to back. A recipe may reuse another recipe's schema only if it says so and names that recipe.

## 4. Safety and design rules (apply to every recipe)

- Human-in-the-loop: anything that reaches a third party (call, SMS, WhatsApp, email send, post publish, booking invite, payment) requires an explicit, action-specific approval unless the recipe states a narrowly bounded pre-approved scope. Drafting is A0/A1; sending is never autonomous by default.
- No silent capture: recording or transcribing people requires consent and a configurable recording-disclosure step that defaults to ON; the runtime must follow local law.
- Honest status: stale is never "healthy"; a tool returning 200 is not proof of the outcome; reconcile ambiguous side effects before retry.
- Least privilege: declare CRUD per resource; unspecified operations are not allowed.
- Secrets: never in recipes — refer to `secret_ref://<name>` style references only.
- Test with rolled-back transactions or sandboxes; never test against real customers, real bookings or real money.
- Do not describe weaknesses of any real deployment; prescribe the secure design.

## 5. Privacy (this package will be published to a third-party marketplace)

- **No real personal or identifying data anywhere:** no real domains (use `example.com`, `example.org`), emails (`jane@example.com`), phone numbers (`+1-555-0100` … `+1-555-0199`), people, companies, account numbers, IP addresses, internal hostnames, chat IDs or file paths that contain usernames. Use fictional data in every example.
- No credentials, tokens, keys, DSNs or webhook URLs — not even partial or redacted ones.

## 6. Length and quality

- Aim for 250–500 lines per `RECIPE.md`: complete, specific and testable; no filler. Every step in Section H has all 14 sub-fields. Section P lists every test the standard names, marking inapplicable ones with a reason. Section M defines all five outcomes.
- Mirror the structure, depth and tone of the reference example `recipes/05-prepare-a-call/` exactly.
