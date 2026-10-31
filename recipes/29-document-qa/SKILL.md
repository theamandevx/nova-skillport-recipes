---
name: nova-document-qa
description: "Skill Port recipe: Document Q&A. The user adds a text document to a private library; the agent splits it into overlapping chunks (about 900 tokens, 150 overlap), stores their embeddings in a pgvector index, and answers questions from the six most similar passages with numbered citations."
---

# Document Q&A

The user adds a text document to a private library; the agent splits it into overlapping chunks (about 900 tokens, 150 overlap), stores their embeddings in a pgvector index, and answers questions from the six most similar passages with numbered citations. When no passage reaches the similarity threshold (about 0.3) it says the question is not covered instead of guessing.

This folder is a **Skill Port recipe** written to Skill Port Recipe Standard v1. It is a **Draft (0.1.0)**: ready for review, not approved for execution. No capability in it is registered with Skill Port, and every binding is marked `UNVERIFIED`.

- **Recipe ID:** `nova.knowledge.document-qa` (proposed until assigned by Skill Port)
- **Version:** 0.1.0 · **Status:** Draft
- **Category:** research
- **Tags:** rag, document-qa, retrieval, embeddings, pgvector, citations, not-covered
- **Vedaxi task type:** none of the seven supported today maps directly (closest: Find research records)
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
