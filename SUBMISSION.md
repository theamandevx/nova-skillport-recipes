# Submitting the Nova recipes to Skill Port

This package holds 52 recipes written to **Skill Port Recipe Standard v1**. Every recipe is a **Draft**: it is ready for review, not approved for execution. Nothing in it has been registered with Skill Port or tested on Vedaxi yet, and each `VALIDATION.md` says so.

## What is in each recipe folder

| File | What it is |
|---|---|
| `SKILL.md` | The package manifest Skill Port's importer requires at the folder root (`name` + `description` front matter), pointing to the files below |
| `RECIPE.md` | The full recipe: sections A–Q of the canonical template, including the capability register |
| `manifest.draft.yaml` | The capability register in the standard's own manifest template (§2.6), marked DRAFT |
| `VALIDATION.md` | The validation report the standard asks for (§7), with the §8 checklist |

There is no JSON file on purpose. Skill Port's JSON Schema is not published, and the standard forbids inventing schema fields.

## Before submitting

1. **Become a verified publisher.** Sign in at Skill Port, create the publisher profile and complete verification. Then replace `Nova (publisher ID TBD …)` in each recipe with the real publisher ID, and change the recipe IDs if Skill Port assigns different ones.
2. **Pick the first batch.** Start with the seven recipes that match a task type Vedaxi supports today:
   `01` Find research records · `02` Remember a note · `03` Find a saved memory · `04` Save a note draft · `05` Prepare a call · `06` Prepare an SMS · `07` Prepare a WhatsApp message.
   Note that `06` is only partly built (no outbound SMS sender yet).
3. **Map the bindings.** Each recipe's capabilities are proposed (`binding://nova/…`, `UNVERIFIED`). With the Vedaxi operator, map each one to a registered binding, or keep it marked as an unresolved dependency.
4. **Fill the TBDs** each `VALIDATION.md` lists: cost caps, retention periods and identity provider.

## How to submit

The standard (§9) says: submit the Markdown recipe and its validation report to Skill Port's review process, plus structured JSON and executable tests when the platform requires them.

Skill Port's published process for skills, which recipes are expected to follow, is:
1. Put the recipe folders in a **public GitHub repository**, one folder per recipe. Each folder already has the `SKILL.md` that Skill Port's package validation requires; without it the import is rejected before scanning.
2. In the Skill Port **Publish** area, use the import wizard to point at each folder.
3. Skill Port packages the folder, records its checksum and runs an **automated scan**. A critical finding blocks it.
4. A **human moderator** approves, rejects or requests changes.
5. Once approved, the recipe appears under Recipes. The Vedaxi operator must still approve the exact recipe version before it can run.

Ask the Skill Port operator to confirm the recipe-specific steps, since the public docs currently describe skills only.

## Privacy

These files are written for a public marketplace. All examples use fictional data (`example.com`, `+1-555-01xx`, "Jane Doe"). There are no credentials, real domains, emails, phone numbers or names of real people or businesses in them, and every file was checked against the owner's real secret values before packaging.
