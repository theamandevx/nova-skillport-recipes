# Lead sourcing

## A. Identity and ownership

- **Recipe ID:** `nova.sales.lead-sourcing` (proposed until assigned by Skill Port)
- **Name:** Lead sourcing
- **Version:** `0.1.0`
- **Status:** Draft
- **Owner or publisher:** Nova (publisher ID TBD — to be assigned at Skill Port publisher verification)
- **Category:** marketing
- **Tags:** sales, prospecting, ideal-customer-profile, metasearch, enrichment, email-verification, review-queue, no-contact
- **Summary:** From an ideal-customer description, the agent finds matching companies through metasearch, reads their public websites to enrich firmographics and a fit score, finds business contacts through an email-finder provider and verifies every address, then stores the result as a list for the owner's review. Nothing and no one is contacted.
- **Changelog:** 0.1.0 — Initial draft converted from the Nova reference implementation.
- **Dependencies and compatibility:** A self-hosted metasearch engine (reference: SearXNG); a polite web fetcher; a structured-output LLM; an email-finder (domain search) provider and an email-verification provider, each with a monthly credit quota; a sales store with leads, a review status and a suppression list (reference: PostgreSQL behind security-definer functions, written through an authenticated ingest gateway); the owner notification channel. All bindings are UNVERIFIED on Skill Port / Vedaxi (see Section B).
- **Evidence status:** Partially verified — reference implementation status: Built. In Nova's reference implementation the pipeline (metasearch discovery → page fetch + LLM firmographic enrichment → provider domain search → provider verification → store) ran end to end at near-zero cost and produced leads with verified business emails. The review gate, compliance tags and quota pre-check are formalised here and were partly manual in the reference. Nothing has been verified on Skill Port or Vedaxi.

## B. Capability register and callable objects

### Registry-confirmed objects

None. No object in this recipe is registered with Skill Port yet.

### Proposed objects (not registry-confirmed)

| Object ID | Type | Purpose | Risk | Status |
|---|---|---|---|---|
| `nova.research.metasearch` | Capability | Web discovery through a self-hosted metasearch engine | low | UNVERIFIED |
| `nova.research.metasearch.search` | Operation (read, external) | Return result URLs and titles for a query | low | UNVERIFIED |
| `nova.web.fetch` | Capability | Polite retrieval of public web pages | low | UNVERIFIED |
| `nova.web.fetch.page` | Operation (read, external) | Fetch one public page, honouring robots rules and rate limits | low | UNVERIFIED |
| `nova.sales.enrich` | Capability | Firmographic extraction from page text | medium | UNVERIFIED |
| `nova.sales.enrich.firmographics` | Operation (read, LLM) | Extract company facts with evidence quotes, and an ICP fit score with reasons | medium | UNVERIFIED |
| `nova.email.discovery` | Capability | Find business contacts for a company domain | medium | UNVERIFIED |
| `nova.email.discovery.domain_search` | Operation (read, external, metered) | Return named contacts and role addresses at a domain | medium | UNVERIFIED |
| `nova.email.verify` | Capability | Check whether an address can receive mail | medium | UNVERIFIED |
| `nova.email.verify.check` | Operation (read, external, metered) | Return `valid`, `accept_all`, `invalid` or `unknown` | medium | UNVERIFIED |
| `nova.sales.sourcing` | Capability | Quotas, review storage and the review list | medium | UNVERIFIED |
| `nova.sales.sourcing.quota_status` | Operation (read) | Remaining provider credits and LLM budget for the period | low | UNVERIFIED |
| `nova.sales.sourcing.store_for_review` | Operation (create/update) | Store companies and contacts as leads with `review_status = pending`, `contactable = false`, deduplicated | medium | UNVERIFIED |
| `nova.sales.suppression` | Capability | Opt-outs and do-not-contact list | medium | UNVERIFIED |
| `nova.sales.suppression.check` | Operation (read) | Is this address or domain suppressed or already a lead? | low | UNVERIFIED |
| `nova.notify.owner` | Capability | Deliver a message to the owner's own channel | low | UNVERIFIED |
| `nova.notify.owner.send` | Operation (external side effect, owner only) | Tell the owner the review list is ready | low | UNVERIFIED |

**Variants (all `health: unverified`, `test_refs: TBD`):**

- `nova.research.metasearch.search.self-hosted` — a self-hosted metasearch instance; no API key; results aggregated from public search engines under the instance's configuration. Only this variant is supported.
- `nova.web.fetch.page.static` — plain HTTP fetch of public pages (home, about, contact, team), identifying user agent, robots rules honoured, at most 1 request per host every 5 seconds, no login, no paywall, no CAPTCHA solving, no JavaScript rendering. Only this variant is supported; JavaScript-heavy sites are reported as `not_readable` rather than rendered.
- `nova.sales.enrich.firmographics.default` — structured-output LLM, temperature 0; every extracted fact must carry a short evidence quote from the fetched text, otherwise it is dropped.
- `nova.email.discovery.domain_search.provider-api` — an email-finder provider's domain search. Only this variant is supported. Scraping social networks or guessing address patterns without verification are not variants of this recipe.
- `nova.email.verify.check.provider-api` — an email-verification provider. Only this variant is supported; SMTP probing from the operator's own infrastructure is excluded (reputation and terms risk).
- All other operations — only the default variant is supported.

**Bindings:** `binding://nova/research/metasearch`, `binding://nova/web/fetch`, `binding://nova/sales/enrich`, `binding://nova/email/domain-search`, `binding://nova/email/verify`, `binding://nova/sales/sourcing-quota`, `binding://nova/sales/store-for-review`, `binding://nova/sales/suppression-check`, `binding://nova/notify/owner` — all `status: UNVERIFIED`. Provider keys as `secret_ref://email-finder`, `secret_ref://email-verifier`, `secret_ref://llm`.

**Resources / object types:**

| Object | Schema | Owner / scope | Operations requested |
|---|---|---|---|
| ICP request | `schema://nova/lead-sourcing/icp-request@0.1.0` | owner | read |
| Company candidate | `schema://nova/lead-sourcing/company@0.1.0` | run-scoped until stored | create (in memory), read |
| Contact candidate | `schema://nova/lead-sourcing/contact@0.1.0` | run-scoped until stored | create (in memory), read |
| Lead (pending review) | `schema://nova/lead-sourcing/review-lead@0.1.0` | owner's tenant | create, update (enrichment of pending leads only) |
| Suppression entry | `schema://nova/lead-sourcing/suppression-result@0.1.0` | owner's tenant | read |

**Authorization requirements (declared, never a grant):** `research:search`, `web:fetch:public`, `sales:enrich`, `email:discovery:search`, `email:verify:check`, `sales:sourcing:write`, `sales:suppression:read`, `notify:owner`.

**Policy / trust:** risk class `medium` — personal data of third parties (business contact details) is collected and stored. No contact is made. The owner is the data controller and must have a lawful basis (for example a documented legitimate-interest assessment) before any later outreach.

## C. Intent and applicability

- **User problem:** The owner needs a steady list of well-matched prospects with real, deliverable business addresses, without buying lists or scraping social networks.
- **Desired outcome:** A reviewed-ready list of up to N companies matching the ideal-customer profile, each with sourced firmographics, a fit score with reasons, up to two verified decision-maker contacts and compliance flags.
- **Trigger or entry conditions:** The owner asks ("find 20 logistics SMEs in the Netherlands with 20–200 staff, COO or founder"), or a scheduled sourcing run with a saved ICP.
- **Applicable situations:** B2B prospecting from public company websites.
- **Out-of-scope situations:** Contacting anyone; enrolling leads in campaigns (a separate, approval-gated recipe); consumer (B2C) prospecting; scraping social networks or sites whose terms forbid automated access; buying or importing third-party lists; sensitive personal data.
- **Required inputs:** ICP request (Section D).
- **Prerequisites:** Metasearch instance reachable; provider accounts with remaining credits; the owner's own domains and existing customers configured as exclusions; the operator's list of opt-in jurisdictions configured.
- **Required capabilities:** listed in Section B (all proposed, UNVERIFIED).
- **Unsupported or missing capabilities:** None registered on Skill Port; every binding is a dependency to resolve before activation. JavaScript rendering is not supported.
- **Assumptions:** Verified in the reference implementation: discovery, enrichment, domain search and verification at near-zero cost; filters for job boards, directories and listicles. Assumed: provider free tiers remain available; the operator maintains the jurisdiction list (this recipe does not encode law).
- **Ambiguity policy:** Ask. If industry, geography or role is missing or too broad to search meaningfully, ask one clarifying question; never widen the ICP silently.
- **Vedaxi task type:** none — not one of the seven task types supported today.

## D. Inputs and outputs

**Inputs**

| Name | Type / format | Required | Source / sensitivity | Default | When missing, invalid, stale or contradictory |
|---|---|---|---|---|---|
| `industries` | array of strings, 1–5 | yes | owner | — | ask |
| `geographies` | array of ISO 3166-1 alpha-2 codes, 1–5 | yes | owner | — | ask; unknown code rejected |
| `roles` | array of job-title phrases, 1–6 | yes | owner | — | ask |
| `company_size` | `{min,max}` employees | no | owner | any | min > max: ask |
| `keywords` | array of strings ≤ 10 | no | owner | none | — |
| `exclude_domains` | array of domains | no | owner | own domains + existing customers (always added) | — |
| `limit` | integer 1–50 companies | no | owner | 20 | clamp |
| `contacts_per_company` | integer 1–2 | no | owner | 2 | clamp |

```json
{"$id":"schema://nova/lead-sourcing/icp-request@0.1.0","type":"object","required":["industries","geographies","roles"],"properties":{"industries":{"type":"array","items":{"type":"string","minLength":2},"minItems":1,"maxItems":5},"geographies":{"type":"array","items":{"type":"string","pattern":"^[A-Z]{2}$"},"minItems":1,"maxItems":5},"roles":{"type":"array","items":{"type":"string","minLength":2},"minItems":1,"maxItems":6},"company_size":{"type":"object","properties":{"min":{"type":"integer","minimum":1},"max":{"type":"integer","minimum":1}}},"keywords":{"type":"array","items":{"type":"string"},"maxItems":10},"exclude_domains":{"type":"array","items":{"type":"string"}},"limit":{"type":"integer","minimum":1,"maximum":50},"contacts_per_company":{"type":"integer","minimum":1,"maximum":2}}}
```

**Outputs**

| Name | Type | Required fields | Destination | Sensitivity | Verification |
|---|---|---|---|---|---|
| `review_list` | `review-list@0.1.0` | `run_id`, `status`, `companies`, `quota_used` | stored leads (pending) + owner's review view | third-party business contact data; owner only | provenance and verification status per field (Section I) |

```json
{"$id":"schema://nova/lead-sourcing/review-list@0.1.0","type":"object","required":["run_id","status","companies","quota_used"],"properties":{"run_id":{"type":"string"},"status":{"enum":["completed","partially_completed","blocked","failed","cancelled"]},"companies":{"type":"array","maxItems":50,"items":{"$ref":"schema://nova/lead-sourcing/company@0.1.0"}},"dropped":{"type":"object","additionalProperties":{"type":"integer"}},"quota_used":{"type":"object","properties":{"domain_searches":{"type":"integer"},"verifications":{"type":"integer"},"llm_tokens":{"type":"integer"}}},"contacted":{"const":0}}}
```

```json
{"$id":"schema://nova/lead-sourcing/company@0.1.0","type":"object","required":["domain","name","country","fit_score","fit_reasons","sources","contacts","flags"],"properties":{"domain":{"type":"string"},"name":{"type":"string"},"country":{"type":"string","pattern":"^[A-Z]{2}$"},"industry":{"type":"string"},"employee_range":{"type":"string"},"description":{"type":"string","maxLength":300},"fit_score":{"type":"integer","minimum":0,"maximum":100},"fit_reasons":{"type":"array","items":{"type":"string"},"maxItems":5},"sources":{"type":"array","items":{"type":"object","required":["url","fetched_at"],"properties":{"url":{"type":"string","format":"uri"},"fetched_at":{"type":"string","format":"date-time"}}}},"contacts":{"type":"array","maxItems":2,"items":{"$ref":"schema://nova/lead-sourcing/contact@0.1.0"}},"flags":{"type":"array","items":{"enum":["opt_in_jurisdiction","consent_regime_unknown","no_verified_contact","role_address_only","competitor","existing_customer","low_fit"]}}}}
```

Other schemas:

```json
{"$id":"schema://nova/lead-sourcing/contact@0.1.0","type":"object","required":["email","verification","role_address","source"],"properties":{"name":{"type":["string","null"]},"title":{"type":["string","null"]},"email":{"type":"string","format":"email"},"verification":{"enum":["valid","accept_all","invalid","unknown"]},"role_address":{"type":"boolean"},"source":{"const":"email_finder_provider"},"send_eligible_after_review":{"type":"boolean"}}}
{"$id":"schema://nova/lead-sourcing/search-query@0.1.0","type":"object","required":["q"],"properties":{"q":{"type":"string","maxLength":200},"page":{"type":"integer","minimum":1,"maximum":3}}}
{"$id":"schema://nova/lead-sourcing/search-results@0.1.0","type":"object","required":["results"],"properties":{"results":{"type":"array","items":{"type":"object","required":["url","title"],"properties":{"url":{"type":"string","format":"uri"},"title":{"type":"string"},"snippet":{"type":"string"}}}}}}
{"$id":"schema://nova/lead-sourcing/fetch-request@0.1.0","type":"object","required":["url"],"properties":{"url":{"type":"string","format":"uri"},"max_bytes":{"type":"integer","maximum":2000000}}}
{"$id":"schema://nova/lead-sourcing/page@0.1.0","type":"object","required":["url","status","fetched_at"],"properties":{"url":{"type":"string"},"status":{"enum":["ok","robots_disallowed","not_readable","not_found","blocked","error"]},"text":{"type":"string","maxLength":60000},"fetched_at":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/lead-sourcing/enrich-request@0.1.0","type":"object","required":["domain","pages","icp"],"properties":{"domain":{"type":"string"},"pages":{"type":"array","items":{"$ref":"schema://nova/lead-sourcing/page@0.1.0"}},"icp":{"$ref":"schema://nova/lead-sourcing/icp-request@0.1.0"}}}
{"$id":"schema://nova/lead-sourcing/firmographics@0.1.0","type":"object","required":["is_company_site","facts","fit_score","fit_reasons"],"properties":{"is_company_site":{"type":"boolean"},"facts":{"type":"array","items":{"type":"object","required":["field","value","evidence"],"properties":{"field":{"enum":["name","country","industry","employee_range","description","offering"]},"value":{"type":"string"},"evidence":{"type":"string","maxLength":200}}}},"fit_score":{"type":"integer","minimum":0,"maximum":100},"fit_reasons":{"type":"array","items":{"type":"string"},"maxItems":5}}}
{"$id":"schema://nova/lead-sourcing/domain-query@0.1.0","type":"object","required":["domain","roles"],"properties":{"domain":{"type":"string"},"roles":{"type":"array","items":{"type":"string"}},"limit":{"type":"integer","minimum":1,"maximum":10}}}
{"$id":"schema://nova/lead-sourcing/domain-contacts@0.1.0","type":"object","required":["contacts"],"properties":{"contacts":{"type":"array","items":{"type":"object","required":["email"],"properties":{"email":{"type":"string"},"name":{"type":["string","null"]},"title":{"type":["string","null"]},"type":{"enum":["personal","generic"]}}}},"credits_used":{"type":"integer"}}}
{"$id":"schema://nova/lead-sourcing/verify-request@0.1.0","type":"object","required":["email"],"properties":{"email":{"type":"string","format":"email"}}}
{"$id":"schema://nova/lead-sourcing/verify-result@0.1.0","type":"object","required":["email","status"],"properties":{"email":{"type":"string"},"status":{"enum":["valid","accept_all","invalid","unknown"]},"credits_used":{"type":"integer"}}}
{"$id":"schema://nova/lead-sourcing/quota-query@0.1.0","type":"object","properties":{"period":{"enum":["current_month"]}}}
{"$id":"schema://nova/lead-sourcing/quota@0.1.0","type":"object","required":["domain_searches_left","verifications_left"],"properties":{"domain_searches_left":{"type":"integer","minimum":0},"verifications_left":{"type":"integer","minimum":0},"llm_tokens_left":{"type":["integer","null"]},"resets_at":{"type":"string","format":"date-time"}}}
{"$id":"schema://nova/lead-sourcing/review-lead@0.1.0","type":"object","required":["dedupe_key","company","review_status","contactable","source","run_id"],"properties":{"dedupe_key":{"type":"string"},"company":{"$ref":"schema://nova/lead-sourcing/company@0.1.0"},"review_status":{"const":"pending"},"contactable":{"const":false},"source":{"const":"sourced"},"run_id":{"type":"string"}}}
{"$id":"schema://nova/lead-sourcing/store-result@0.1.0","type":"object","required":["created","updated","skipped_existing"],"properties":{"created":{"type":"integer"},"updated":{"type":"integer"},"skipped_existing":{"type":"integer"}}}
{"$id":"schema://nova/lead-sourcing/suppression-query@0.1.0","type":"object","properties":{"email":{"type":"string"},"domain":{"type":"string"}}}
{"$id":"schema://nova/lead-sourcing/suppression-result@0.1.0","type":"object","required":["suppressed","existing_lead"],"properties":{"suppressed":{"type":"boolean"},"existing_lead":{"type":"boolean"},"reason":{"type":["string","null"]}}}
{"$id":"schema://nova/lead-sourcing/owner-note@0.1.0","type":"object","required":["text"],"properties":{"text":{"type":"string","maxLength":1500}}}
{"$id":"schema://nova/lead-sourcing/delivery@0.1.0","type":"object","required":["delivered"],"properties":{"delivered":{"type":"boolean"},"message_id":{"type":"string"}}}
```

**Example (fictional)**

Input: `{"industries":["freight forwarding"],"geographies":["NL"],"roles":["founder","COO"],"company_size":{"min":20,"max":200},"limit":10}`

Output (one company): `{"domain":"example.com","name":"Example Freight BV","country":"NL","industry":"freight forwarding","employee_range":"50-100","fit_score":82,"fit_reasons":["freight forwarder in the target country","team page shows ~60 staff","runs manual booking forms (automation need)"],"sources":[{"url":"https://example.com/about","fetched_at":"2026-03-05T10:12:00Z"}],"contacts":[{"name":"Jane Doe","title":"COO","email":"jane@example.com","verification":"valid","role_address":false,"source":"email_finder_provider","send_eligible_after_review":true}],"flags":[]}` — run total `contacted: 0`.

## E. Constraints and budgets

- **Hard constraints:** contact no one (no email, message, connection request or call); store leads only as `review_status = pending`, `contactable = false`; only public pages, honouring robots rules, without logins, paywalls or CAPTCHA solving; no social-network scraping; every fact carries a source URL and an evidence quote; every contact address is verified by the provider before storage, and `invalid` addresses are discarded; role addresses are flagged and never marked send-eligible; suppressed addresses and domains, the owner's own domains and existing customers are excluded; companies in operator-listed opt-in jurisdictions are flagged `opt_in_jurisdiction` and never marked send-eligible; store only business contact data (name, title, business email, company).
- **Soft constraints:** prefer named decision-makers over generic addresses; prefer higher fit scores; diversify across the requested geographies.
- **Resource and tenant scope:** the owner's leads and suppression list.
- **Data sources allowed:** metasearch results, public company websites, the two providers' responses.
- **External transfers permitted:** search queries to the metasearch engine (and through it to public search engines); company domains to the email-finder; candidate addresses to the verifier; page text to the LLM provider.
- **Time limit:** run ≤ 45 minutes.
- **Cost or token budget:** provider credits per run ≤ the remaining monthly quota and ≤ a per-run cap (`TBD`, operator; the reference ran on free tiers of roughly 50 domain searches and 100 verifications per month); LLM token cap per run `TBD`.
- **Maximum steps:** 10 (S1–S10); S5–S8 repeat per company.
- **Maximum retries and correction cycles:** search and fetch 2 retries; LLM 1 retry; provider calls 1 retry on transient errors only (credits are not spent twice on a definite answer); 1 query-broadening cycle, only within the ICP.
- **Rate limits and concurrency:** 1 request per host per 5 seconds; provider rate limits; at most 3 companies processed in parallel.
- **Retention and logging constraints:** pending leads that the owner neither approves nor rejects are deleted after `TBD` days (operator; default proposal 90); rejected leads keep only a dedupe key so they are not re-sourced; logs carry domains and counts, never addresses.
- **Prohibited actions:** any outreach; enrolment in campaigns; marking leads contactable; storing unverified or invalid addresses; inferring personal data beyond business role.
- **Stop conditions:** quota insufficient for even one company; metasearch unreachable; owner cancels.

## F. CRUD and side effects

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | lead with `review_status = pending`, `contactable = false`, `source = sourced` | passed S4–S8; not suppressed or existing | Allowed | stored row matches `review-lead@0.1.0` |
| Read | ICP, suppression list, existing leads (dedupe), quotas | own tenant | Allowed | — |
| Read | public web pages, provider responses | public data, metered | Allowed | provenance recorded |
| Update | enrichment of an existing *pending* sourced lead (same dedupe key) | lead still pending | Allowed | only enrichment fields changed |
| Update | `review_status`, `contactable`, stage, enrolment of any lead | — | Not Allowed (owner action in the review view) | — |
| Delete | any lead | — | Not Allowed (retention job is separate) | — |
| External side effect | metered provider calls | within quota and per-run cap | Allowed | credits used reported |
| External side effect | any contact with a prospect | — | Not Allowed | contacted = 0 |
| External side effect | owner note | list ready | Allowed | delivery receipt |

- **Idempotency:** dedupe key = company domain (and, per contact, normalised email); re-running the same ICP updates pending leads and skips leads already reviewed.
- **Expected side effects:** pending leads stored; provider credits spent; LLM tokens billed.
- **Compensation:** the owner rejects leads in the review view; rejected leads are reduced to a dedupe key.
- **Irreversible:** spent provider credits; queries sent to external services.
- **Ambiguous outcomes:** if a provider call times out, check the provider's usage counter (if available) before retrying, and never retry more than once.

## G. Autonomy and approval policy

- **Default autonomy level:** A2 — bounded autonomy for research; no outreach.
- **Planning level:** A2.
- **Execution level:** A2 within the ICP, quota and per-run cap.
- **Verification level:** A2.
- **Recovery level:** A2 — retries and one ICP-bounded query broadening.
- **Approval required for:** exceeding the per-run credit cap; any paid-tier usage beyond the free quota; adding a new data source or provider.
- **Actions never permitted:** contacting prospects; marking leads contactable; enrolling leads; social-network scraping; bypassing robots rules, logins or CAPTCHAs.
- **Conditions requiring clarification:** missing or overly broad ICP fields.
- **Conditions requiring escalation:** quota exhausted; a provider changes terms or starts returning errors persistently.
- **Safe state after stop, cancellation or escalation:** whatever is already verified is stored as pending; nothing contacted; partial results disclosed.

## H. Execution plan

**S1 — Validate the ICP request**
1. Objective: a precise, searchable ICP.
2. Capability: local validation.
3. Input mapping: owner message → `icp-request@0.1.0`; own domains and existing customers added to exclusions.
4. Dependencies: none.
5. Preconditions: owner request or saved ICP.
6. Action / side effects: validate; no side effects.
7. Autonomy: A2; no approval.
8. Timeout / retry: 5 s / none.
9. Idempotency: not applicable.
10. Postconditions: valid ICP.
11. Verification: JSON Schema passes; at least one industry, geography and role.
12. Failure routes: missing or invalid → ask once → else `blocked`.
13. Escalation triggers: none.
14. Evidence: validated ICP.

**S2 — Check quota and budget**
1. Objective: never start a run the budget cannot finish honestly.
2. Capability: `nova.sales.sourcing.quota_status`.
3. Input mapping: current period.
4. Dependencies: S1.
5. Preconditions: ICP valid.
6. Action / side effects: read remaining domain searches, verifications and LLM budget.
7. Autonomy: A2.
8. Timeout / retry: 10 s / 2 retries.
9. Idempotency: read-only.
10. Postconditions: an effective company limit = min(`limit`, domain searches left, per-run cap, verifications left ÷ `contacts_per_company`).
11. Verification: effective limit ≥ 1.
12. Failure routes: effective limit 0 → `blocked` with the reset date; lower than requested → continue and disclose.
13. Escalation triggers: paid tier needed → ask the owner (Section L).
14. Evidence: quota snapshot.

**S3 — Build queries and search**
1. Objective: a broad but on-target set of candidate company sites.
2. Capability: `nova.research.metasearch.search.self-hosted`.
3. Input mapping: industries × geographies × keywords → up to 12 queries (for example "freight forwarding company Netherlands").
4. Dependencies: S2.
5. Preconditions: metasearch reachable.
6. Action / side effects: queries to the metasearch engine; up to 3 result pages each.
7. Autonomy: A2.
8. Timeout / retry: 20 s per query / 2 retries.
9. Idempotency: read-only.
10. Postconditions: candidate URLs reduced to registrable domains.
11. Verification: at least 1 result overall.
12. Failure routes: no results → one broadening cycle within the ICP (drop keywords, add synonyms of the industry) → else `completed` with zero companies and a note.
13. Escalation triggers: metasearch unreachable → `blocked`.
14. Evidence: queries and result counts.

**S4 — Filter and de-duplicate candidates**
1. Objective: keep only plausible company websites not already known.
2. Capability: local filter rules; `nova.sales.suppression.check`.
3. Input mapping: candidate domains, exclusions.
4. Dependencies: S3.
5. Preconditions: candidates present.
6. Action / side effects: drop job boards, business directories, marketplaces, social networks, news and listicle pages ("top 10 …", "best … companies"), government sites, the owner's domains, existing customers, suppressed domains and existing leads; keep the first `effective limit × 2` candidates for enrichment.
7. Autonomy: A2.
8. Timeout / retry: 10 s per check / 2 retries.
9. Idempotency: read-only.
10. Postconditions: a shortlist of unique domains with drop counts by reason.
11. Verification: no excluded domain survives.
12. Failure routes: suppression check unavailable → `blocked` (never assume not suppressed).
13. Escalation triggers: none.
14. Evidence: shortlist and drop counts.

**S5 — Fetch and enrich (per company)**
1. Objective: sourced firmographics and an ICP fit score.
2. Capability: `nova.web.fetch.page.static`; `nova.sales.enrich.firmographics`.
3. Input mapping: home, about, contact and team pages of the domain; ICP.
4. Dependencies: S4.
5. Preconditions: robots rules allow the pages.
6. Action / side effects: polite fetches; LLM extraction with evidence quotes; page text treated as untrusted data.
7. Autonomy: A2.
8. Timeout / retry: 20 s per page / 2 retries; LLM 90 s / 1 retry.
9. Idempotency: read-only.
10. Postconditions: `firmographics@0.1.0`.
11. Verification: `is_company_site = true`; every fact's evidence quote appears in the fetched text; country is in the requested geographies.
12. Failure routes: robots disallowed, not readable or not a company site → drop with the reason; facts without evidence → removed.
13. Escalation triggers: none.
14. Evidence: source URLs, fetch times, fact-evidence pairs.

**S6 — Score fit and tag compliance**
1. Objective: rank companies and flag what limits later contact.
2. Capability: local rules over S5 output and the operator's jurisdiction list.
3. Input mapping: firmographics, ICP size band, country.
4. Dependencies: S5.
5. Preconditions: enrichment passed.
6. Action / side effects: keep fit ≥ 50 (configurable); add `low_fit` below; add `opt_in_jurisdiction` or `consent_regime_unknown` by country; add `competitor` when the offering overlaps the owner's.
7. Autonomy: A2.
8. Timeout / retry: 2 s / none.
9. Idempotency: deterministic.
10. Postconditions: ranked companies up to the effective limit.
11. Verification: every company has fit reasons drawn from evidenced facts.
12. Failure routes: fewer than requested → continue and disclose.
13. Escalation triggers: none.
14. Evidence: scores and flags.

**S7 — Discover contacts (per qualified company)**
1. Objective: named decision-makers at the company domain.
2. Capability: `nova.email.discovery.domain_search.provider-api`.
3. Input mapping: domain, roles, limit 10.
4. Dependencies: S6.
5. Preconditions: domain-search credits left.
6. Action / side effects: one metered provider call per company.
7. Autonomy: A2.
8. Timeout / retry: 20 s / 1 retry on transient error only.
9. Idempotency: one call per domain per run.
10. Postconditions: up to `contacts_per_company` candidates, role-matched first; generic addresses kept only if no named contact exists and flagged `role_address`.
11. Verification: every address is at the company's domain; credits counted.
12. Failure routes: no contacts → company kept with `no_verified_contact`; quota hit → stop discovery, continue to S9 with what exists.
13. Escalation triggers: none.
14. Evidence: credits used, contact count (not addresses in logs).

**S8 — Verify every address**
1. Objective: only deliverable addresses survive.
2. Capability: `nova.email.verify.check.provider-api`; `nova.sales.suppression.check`.
3. Input mapping: each candidate address.
4. Dependencies: S7.
5. Preconditions: verification credits left; address not suppressed.
6. Action / side effects: one metered call per address.
7. Autonomy: A2.
8. Timeout / retry: 20 s / 1 retry on transient error only.
9. Idempotency: one call per address per run; cached result reused within 30 days.
10. Postconditions: `valid` kept and `send_eligible_after_review = true` unless a role address or the company is flagged opt-in or unknown; `accept_all` and `unknown` kept with `send_eligible_after_review = false`; `invalid` discarded.
11. Verification: no `invalid` address stored; every stored address has a verification status.
12. Failure routes: quota hit → remaining contacts discarded (unverified addresses are never stored) and disclosed.
13. Escalation triggers: none.
14. Evidence: verification statuses and credits used.

**S9 — Store for review**
1. Objective: the list lands in the owner's review queue, contactable by no one.
2. Capability: `nova.sales.sourcing.store_for_review`.
3. Input mapping: companies with contacts and flags → `review-lead@0.1.0`.
4. Dependencies: S8.
5. Preconditions: at least one company.
6. Action / side effects: create or update pending leads, deduplicated by domain and email; leads already reviewed are skipped.
7. Autonomy: A2.
8. Timeout / retry: 15 s / 2 retries.
9. Idempotency: dedupe key; repeat calls update the same pending rows.
10. Postconditions: `store-result@0.1.0`.
11. Verification: re-read: every stored lead has `review_status = pending` and `contactable = false`.
12. Failure routes: store error → keep the review list in the run record; `partially_completed`.
13. Escalation triggers: none.
14. Evidence: lead IDs.

**S10 — Deliver the review list**
1. Objective: the owner knows the list is ready and what limited it.
2. Capability: `nova.notify.owner.send`.
3. Input mapping: counts, top companies by fit, drop reasons, quota used, review-view link.
4. Dependencies: S9.
5. Preconditions: run finished or stopped.
6. Action / side effects: one owner note ("10 companies, 14 verified contacts ready for review; 0 contacted").
7. Autonomy: A2.
8. Timeout / retry: 15 s / 2 retries.
9. Idempotency: note key = `run_id`.
10. Postconditions: delivered.
11. Verification: delivery receipt.
12. Failure routes: undelivered → the list stays in the review view; `partially_completed`.
13. Escalation triggers: none.
14. Evidence: message ID.

## I. Continuous verification

| Check | Assertion | Evidence source | Pass / fail rule | On failure |
|---|---|---|---|---|
| ICP validity | matches `icp-request@0.1.0` | validator | valid | ask, then `blocked` |
| Budget | provider credits used ≤ quota and per-run cap | quota snapshot vs usage | within | stop metered calls |
| Polite access | robots rules honoured; ≤ 1 request per host per 5 s | fetch log | holds | stop fetching that host |
| Provenance | every fact has a URL, time and evidence quote found in the text | S5 | 100 % | drop the fact |
| Exclusions | no own, customer, suppressed or existing domain | S4 vs lists | none | drop, report a rule bug |
| Verification | no stored address is `invalid` or unverified | S8, stored rows | none | remove before storage |
| Send-eligibility | false for role, accept-all, unknown, opt-in or unknown-regime | stored rows | holds | correct flag |
| No contact | outbound messages to prospects = 0 | outbound logs | 0 | stop, escalate |
| Review state | all stored leads pending and not contactable | S9 re-read | holds | fix flags; report |

A provider saying `valid` is a probability, not proof of a human reader; the owner's review remains the gate before any outreach.

## J. Self-correction and recovery

- **Failure classes:** invalid ICP, quota exhausted, metasearch down, page not readable, robots disallowed, LLM error, unsupported fact, provider transient error, provider definite error, store error.
- **Permitted corrections:** one ICP-bounded query broadening; retries on transient errors; dropping unreadable sites; removing unsupported facts.
- **Forbidden corrections:** widening industries, geographies or roles without the owner; guessing addresses; storing unverified addresses; switching to social-network scraping; exceeding the credit cap.
- **Maximum correction cycles:** 1 broadening per run; 1 retry per external call.
- **Reverification required after correction:** yes — S4 filters and S5 evidence checks apply to broadened results.
- **Stop conditions:** quota exhausted; metasearch unreachable; owner cancels.
- **Recovery or compensation procedure:** partial lists are stored as pending; the owner rejects unwanted leads.
- **What must be reported to the user:** requested vs delivered counts, why companies were dropped, credits used and remaining, and that nobody was contacted.

## K. Independent challenge

- **Assumptions to challenge:** that a website's claims (size, industry) are current; that a named contact still works there; that "fits the ICP" means "has the problem the owner solves".
- **Alternative approach:** for a small, high-value list, manual research by the owner may beat volume; offer `limit ≤ 10` with deeper enrichment.
- **Contradictory evidence:** company pages that list a different country or a much larger headcount than the ICP; `accept_all` domains where every address "verifies".
- **Could mislead:** a high fit score built on one thin fact; a list that looks complete but was cut short by quota.
- **Separate verifier:** a rule check recomputes the fit score from evidenced facts only and flags gaps over 20 points from the LLM score.
- **Findings requiring disclosure:** quota truncation, many `accept_all` results, companies with no verified contact.

## L. Human escalation

- **Triggers:** quota exhausted or paid tier needed; persistent provider errors or terms changes; an outbound message to a prospect detected (should be impossible).
- **Why the agent cannot decide:** spending money and data-protection exposure are the owner's decisions.
- **Information presented:** quota used and remaining, reset date, cost of the next tier as reported by the provider, partial results.
- **Decision requested:** wait for reset, approve paid usage for this run, or narrow the ICP.
- **Options and consequences:** waiting delays the list; paying spends money; narrowing yields fewer but better matches.
- **Safe behaviour while waiting:** no metered calls; partial list stored as pending.
- **If no response:** the run ends `partially_completed`; the next scheduled run starts after the quota resets.
- **Resume:** S2 re-reads the quota before any metered call.

## M. Completion conditions

Mandatory: review list valid; every stored lead pending and not contactable; every stored address verified and not invalid; provenance present for every fact; nobody contacted; quota use reported; limitations disclosed.

- `completed` — the requested number of companies (or every qualifying company found) stored for review, within budget.
- `partially_completed` — fewer companies than requested because of quota, filters, unreadable sites or a store or notification failure.
- `blocked` — invalid ICP, zero quota, metasearch unreachable or suppression check unavailable.
- `failed` — the pipeline errored before any company could be stored.
- `cancelled` — the owner stopped the run; verified results so far are stored as pending.

## N. Final output contract

Return: run ID; recipe ID and version; status; one-line outcome ("10 companies, 14 verified contacts for review; 0 contacted"); `review-list@0.1.0`; step statuses S1–S10; verification results and failed checks; evidence references (source URLs, verification statuses, lead IDs); approvals and scope (credit cap); side effects (credits spent, leads stored) and reconciliation status; corrections and retries; unresolved challenges; limitations; escalation state; learning candidate. Never include provider keys; addresses only in the owner's review view.

## O. Learning and revision

- **Outcome metrics:** companies per query, filter drop rates, fit-score distribution, verified-contact rate, `accept_all` share, owner approval rate per run, later reply rate of approved leads (from the outreach recipe), credits per approved lead.
- **Evidence retained:** run records, provenance and verification statuses for stored leads.
- **Data minimisation:** rejected leads reduced to a dedupe key; metrics without addresses.
- **Lesson candidate format:** `{trigger, evidence_ref, proposed_change, test_to_add}`.
- **Review owner:** the publisher (TBD).
- **Tests for a revised recipe:** all Section P tests.
- **Promotion approval:** publisher review plus Skill Port review; any new data source needs a privacy review.

## P. Testing and publication readiness

| Test | Expected behaviour | Evidence status |
|---|---|---|
| Happy path (fictional ICP against a mocked web and mocked providers) | companies stored pending with verified contacts; 0 contacted | UNVERIFIED on Skill Port; pipeline observed in the reference implementation |
| Invalid input (unknown country code) | rejected, asks | TBD |
| Missing input (no roles) | asks once, then `blocked` | TBD |
| Missing capability (verifier binding absent) | contacts discarded, companies stored with `no_verified_contact`, disclosed — no unverified address stored | TBD |
| Permission denied (`sales:sourcing:write` missing) | `blocked` | TBD |
| Unauthorized scope (request to mark leads contactable) | refused | TBD |
| Tenant isolation | dedupe and suppression never read another tenant's data | TBD |
| Robots disallow | site skipped with the reason | TBD |
| Listicle and directory results | filtered out | TBD |
| Suppressed domain | never stored | TBD |
| Quota partially available | effective limit reduced and disclosed | TBD |
| Quota zero | `blocked` with reset date | TBD |
| Timeout on provider call | one retry at most; credits not double-spent | TBD |
| Duplicate / replay (same ICP twice) | pending leads updated, reviewed leads skipped | TBD |
| False success (LLM invents headcount) | fact dropped (no evidence quote) | TBD |
| Failed verification (`invalid` address) | discarded | TBD |
| Correction budget exhausted (no results after broadening) | `completed` with zero and a note | TBD |
| Escalation (paid tier needed) | owner asked; no spend | TBD |
| Cancellation mid-run | verified results stored pending; nothing contacted | TBD |
| Compensation (owner rejects leads) | reduced to dedupe keys; not re-sourced | TBD |
| Output schema and status | valid `review-list`, `contacted = 0` | TBD |
| Compatibility / regression (provider API change) | verification statuses still mapped | TBD |

Tests use mocked web pages and mocked providers with fictional companies at example.com; never real people's data. Not production-ready until the mandatory tests have evidence.

## Q. Open questions and known limitations

- Skill Port / Vedaxi binding IDs and how metered provider credits are surfaced — TBD (owner: publisher, with Skill Port).
- Per-run credit cap, fit threshold and pending-lead retention — TBD (owner: operator).
- The opt-in jurisdiction list is operator configuration; this recipe does not encode law and does not give legal advice.
- JavaScript-heavy websites are reported as not readable; a rendering variant would need its own review.
- Free provider tiers bound volume (tens of companies per month in the reference implementation).
- Verification reduces bounces but cannot prove consent or interest; outreach remains a separate, approval-gated recipe.
