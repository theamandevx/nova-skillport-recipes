# Skill Port Recipe Standard
**Version:** 1.0  
**Status:** Authoring standard  
**Purpose:** A reusable contract for designing, documenting, validating, and submitting recipes to Skill Port.

---

## 1. Purpose and core rule

A **recipe** is a structured capability-register package plus invocation/composition instructions. It identifies callable capabilities, operations, tool bindings, variants, schemas, dependencies, authorization requirements, and how these objects are selected and composed into a workflow with constraints, verification, recovery, and a final output.

Every recipe submitted to Skill Port must follow this standard. Authors may use an LLM to draft or convert an existing workflow, prompt, script, or application process into this format.

**Important:** A recipe describes what an agent should do. It does not itself grant permissions, prove that a capability exists, or authorize an action. The executing platform remains responsible for authentication, authorization, consent, trust, policy enforcement, tenant isolation, and runtime controls.

Use Markdown for the human-readable recipe document. Where the implementation requires machine-readable execution, represent the same contract in the platform's approved structured schema (such as JSON validated against the current JSON Schema). Do not invent fields, capability identifiers, APIs, or platform behavior: mark anything not verified as `TBD` or `UNVERIFIED`.


## 2. Capability-register model (mandatory)

A Skill Port recipe is not merely a prompt or a list of workflow steps. It is a **capability-register package plus invocation/composition instructions**. It declares which callable objects are available, which variants each object supports, how each operation is invoked, what inputs and outputs it accepts, what authorization and dependencies it requires, and how a workflow composes those capabilities.

The recipe may reference existing registered objects or bundle new capability declarations for review. It must not claim that a referenced object exists or is installed until the registry/runtime confirms it.

### 2.1 Registry object model

Use the platform's existing identifiers and schemas where they exist. The following is the conceptual object inventory the recipe authoring format must be able to represent; it is not permission to create duplicate runtime registries.

| Object | Purpose | Minimum information |
|---|---|---|
| **Provider** | Organization or service that supplies an integration | `provider_id`, name, ownership, trust/provenance, status |
| **App / package** | Installable product or integration containing capabilities | `app_id`, version, publisher, description, lifecycle status, compatibility |
| **Capability** | Named ability exposed to the planner/agent | `capability_id`, purpose, input/output contracts, risk, prerequisites, availability |
| **Operation / function** | A specific callable action within a capability | `operation_id`, operation name, description, input schema, output schema, side effects, idempotency |
| **Tool binding** | Concrete invocation target for an operation | `binding_id`, tool/function/API name, provider, protocol, endpoint or registered target reference, invocation mode |
| **Variant** | An explicitly supported implementation or mode of an operation | `variant_id`, parent operation, selection conditions, required inputs, constraints, cost/latency/quality trade-offs, compatibility |
| **Resource / object type** | Entity that an operation reads or mutates | resource type, CRUD permissions requested, scope, ownership, sensitivity, retention |
| **Input/output schema** | Machine-checkable contract for parameters and results | schema ID/version, type definitions, required fields, validation rules, examples |
| **Dependency** | Required capability, app, service, data source, or environment | referenced ID, version constraint, required/optional status, health prerequisite, failure behavior |
| **Installation / binding instance** | Installed, configured instance available to a user or organization | installation ID, app/version, tenant/user scope, enabled state, configured bindings |
| **Authorization requirement** | Scopes/consent/grants needed to call an operation | requested scopes, resource scope, consent requirement, approval requirement, expiry expectations; never a grant itself |
| **Protocol / interface** | Invocation and interoperability contract | API/OpenAPI, MCP tool, A2A interface, event/AsyncAPI, SDK/function or other registered adapter |
| **Policy / trust requirement** | Conditions imposed by runtime security controls | risk class, trust requirements, data handling, prohibited actions, approval and stop rules |
| **Test / verification definition** | How an operation or variant is validated | test ID, fixtures, assertions, expected effects, negative cases, evidence produced |
| **Health / quality record** | Operational and quality status of an object | structural, functional, operational, authorization health; timestamp, evidence, failure rate, known limitations |
| **Version / compatibility record** | Controls changes and supported combinations | semantic version or platform-approved version, compatibility range, deprecation, migration notes |
| **Usage / outcome metric** | Evidence for operational quality and ranking | invocation count, success/failure, latency/cost where available, verification outcome, freshness and provenance |

An object can expose multiple operations; an operation can expose multiple variants; a variant can use one or more registered bindings if the platform explicitly supports that arrangement. Do not flatten these into a single ambiguous `tool` string.

### 2.2 Operation variants

When an operation has multiple variants, list each one explicitly. Examples of variant dimensions include:
- Different registered providers or implementations of the same capability.
- Read-only versus mutation-capable modes.
- Synchronous versus asynchronous invocation.
- Quality, latency, cost, context-size, or reliability trade-offs.
- Environment or version-specific implementations.
- Human-assisted versus agent-executable modes, where the underlying action and permissions genuinely differ.

For every variant specify:
- Stable `variant_id` and parent `operation_id`.
- Exact registered binding(s) and protocol.
- Input/output schema compatibility.
- Preconditions, supported environments, and dependencies.
- Required authorization scopes and side effects.
- Risk classification and approval rules.
- Selection constraints and fallback eligibility.
- Timeout, retry, idempotency, and compensation behavior.
- Verification tests, health status, and evidence references.
- Version compatibility and deprecation status.

Variant selection must be deterministic or explainable: state the eligibility filters and the selection policy (for example, compatible version first, then policy eligibility, then measured quality/cost/latency). Ranking must never override authorization, trust, safety, tenant scope, or explicit user constraints. A fallback variant may be used only if it independently satisfies the same constraints and authorization requirements. Never silently fall back from a read-only variant to one that can mutate data.

### 2.3 Callable-object contract

Each callable operation should document, as applicable:

```yaml
capability:
  capability_id: "provider.capability"
  name: "Capability name"
  description: "Outcome the capability supports"
  status: "draft" # registry-confirmed status must be supplied by the platform
  risk_class: "low"
  operations:
    - operation_id: "provider.capability.operation"
      name: "Operation name"
      description: "Specific action performed"
      input_schema_ref: "schema://operation-input"
      output_schema_ref: "schema://operation-output"
      side_effects: []
      idempotency: "required | supported | not_supported"
      variants:
        - variant_id: "provider.capability.operation.variant-a"
          binding_ref: "registered-binding-id"
          protocol: "MCP | OpenAPI | A2A | SDK | event | other"
          eligibility:
            constraints: []
            environments: []
            version_range: "TBD"
          authorization:
            required_scopes: []
            approval_required: false
          execution:
            mode: "sync | async | event"
            timeout_seconds: 30
            retry_policy: "bounded"
          verification:
            test_refs: []
            evidence_requirements: []
          health:
            status: "unverified"
            evidence_ref: null
```

This is an illustrative authoring shape, not an approved platform schema. Use actual registered IDs, enum values, protocol adapters, and schema references. Do not publish placeholder values as live bindings.

### 2.4 Invocation and composition rules

For each operation, define:
1. How the planner discovers and selects it.
2. How input values are mapped from recipe inputs or prior step outputs.
3. How the runtime resolves the registered binding and installed instance.
4. How authorization is checked at call time.
5. How the operation is invoked through its registered protocol adapter.
6. How structured success, failure, timeout, and retry results are normalized.
7. How outputs and evidence are passed to downstream steps.
8. How idempotency, replay, correlation IDs, and side-effect reconciliation work where applicable.
9. How health and compatibility affect eligibility.
10. How the operation's results update usage/quality metrics without exposing sensitive payloads.

Recipes describe composition and intent; the platform's registered adapter performs the actual invocation. Never place secrets, live credentials, private tokens, or unprotected connection strings in a recipe.

### 2.5 Capability-register health and repair

The authoring and publication process must be able to represent and report:
- **Structural health:** schema validity, required metadata, resolvable references.
- **Functional health:** operation and variant tests pass.
- **Operational health:** endpoint/service availability, timeouts, error rates and freshness of health evidence.
- **Authorization health:** required scopes are defined and actual runtime authorization is enforced.
- **Compatibility health:** app, operation, variant, schema and runtime versions are compatible.
- **Recipe composition health:** dependencies resolve, data mappings type-check, and step contracts align.

Use the quality lifecycle:

**Publish draft → Validate → Test → Execute under policy → Measure → Report → Propose repair → Retest → Review and release.**

An LLM-assisted healing feature may diagnose failures and propose edits or tests. It must show the evidence and diff, preserve the original version, rerun applicable tests, and require the configured approval before promoting a repair. It must not grant permissions, bypass policy, or silently modify a trusted published version.



## 2.6. Required manifest inventory and object graph

In Skill Port, a recipe is not only a workflow narrative or sequence of steps. It is a **versioned capability register**: a machine-readable declaration of the capabilities an app or agent exposes, the callable tools/functions, the objects those tools operate on, the supported variants, their schemas, dependencies, authorization requirements, and execution contracts. The workflow lifecycle in this document describes how those registered capabilities are selected and orchestrated.

A published recipe may expose one capability or a bundle of related capabilities. Every exposed item must be explicitly declared and independently validated. Do not assume that a tool, object, variant, or permission exists merely because it appears in prose.

### Capability-register layers

1. **App/package** — the published app or capability package, owner, version, lifecycle status, compatibility, and installation requirements.
2. **Capability** — a user- or agent-facing ability, its purpose, input/output contracts, preconditions, risk class, and success criteria.
3. **Callable tool/function** — the concrete operation invoked by a runtime or adapter, including its stable identifier, invocation mode, argument schema, result schema, timeout, retry/idempotency behavior, and error contract.
4. **Objects/resources** — the typed entities the capability can read, create, update, delete, list, search, or otherwise act upon, including ownership and scope rules.
5. **Variants** — supported modes or alternatives for a capability/tool/object, such as provider, model, operation, format, execution mode, or resource subtype. Each variant must declare its own compatibility, schema differences, constraints, and fallback policy. Do not infer variants from names.
6. **Bindings/adapters** — the mapping from a declared tool to an actual callable implementation, such as an MCP tool, A2A capability, OpenAPI operation, internal runtime capability, or other explicitly supported connector.
7. **Authorization and installation** — required identity/provider, scopes, user or organization consent, installation binding, secrets reference (never the secret itself), tenant scope, and activation conditions.
8. **Contracts and evidence** — JSON Schemas or equivalent input/output definitions, errors, examples, health checks, test references, provenance, and verification results.

### Required manifest inventory

For every package, list all of its capabilities. For every capability, list all callable tools. For each tool, list the applicable objects and every supported variant. Use stable identifiers and explicit references.

The register must make it possible for the platform to answer:
- What can this package do?
- Which capability provides the requested ability?
- Which exact tool/function will be called?
- Which objects/resources can it affect?
- Which variants are available and compatible in this environment?
- What inputs and outputs are valid?
- What permissions, grants, provider connections, and installation bindings are required?
- What are the side effects, risks, errors, limits, and retry rules?
- How is the call tested, monitored, and verified?
- What happens if the tool or provider is unavailable or a variant becomes incompatible?

### Capability manifest template

Use this as a conceptual authoring shape. The exact property names and accepted enumerations must be reconciled with Skill Port's currently implemented JSON Schema and runtime interfaces before being treated as production schema.

```yaml
manifest:
  package:
    app_id: "publisher.package"
    name: "Package Name"
    version: "1.0.0"
    owner: "publisher-id"
    status: "draft"
    description: "What the package enables"
    compatibility:
      recipe_contract: "1.x"
      runtimes: []
      protocols: []
    installation:
      mode: "user | organization | tenant"
      prerequisites: []
      health_check_ref: null

  capabilities:
    - capability_id: "publisher.package.capability"
      name: "Capability Name"
      description: "Outcome this capability enables"
      intent_examples: []
      input_schema_ref: "schema://capability-input"
      output_schema_ref: "schema://capability-output"
      risk_class: "low | medium | high | critical"
      preconditions: []
      postconditions: []
      tools:
        - tool_id: "publisher.package.tool"
          name: "Callable Tool Name"
          description: "Exact operation performed"
          invocation:
            protocol: "MCP | A2A | OpenAPI | internal | other"
            binding_ref: "binding://actual-implementation"
            operation: "actual-operation-id"
          input_schema_ref: "schema://tool-input"
          output_schema_ref: "schema://tool-output"
          objects:
            - object_id: "publisher.package.object"
              name: "Object Type"
              description: "Resource the tool acts on"
              operations: ["read"]
              scope_rules: []
              schema_ref: "schema://object"
          variants:
            - variant_id: "default"
              description: "Default supported mode"
              compatible_environments: []
              input_schema_ref: "schema://variant-input"
              output_schema_ref: "schema://variant-output"
              constraints: []
              fallback_variant_id: null
          authorization:
            required: true
            identity_provider: "TBD"
            required_scopes: []
            consent_required: true
            grant_ref: null
            tenant_binding_required: true
          execution:
            side_effects: []
            idempotency: "required | recommended | not_applicable"
            timeout_seconds: 30
            max_attempts: 1
            rate_limits: []
            error_codes: []
          verification:
            health_check_ref: null
            test_refs: []
            success_assertions: []
          availability:
            status: "unverified"
            last_verified_at: null
            evidence_refs: []
```

This is an authoring template, not a claim that these exact fields are already accepted by the deployed platform. Do not copy it into production unchanged until the canonical platform schema is identified and validated.

### Objects and variants: mandatory authoring rules

- **Objects must be typed.** Define each resource/object, its schema, ownership, tenant scope, supported operations, and fields that may be read or changed. Do not use vague object names such as “data” when the actual resource type is known.
- **Tools must be callable.** Declare the stable tool/function ID, actual binding, protocol or adapter, operation name, argument schema, result schema, timeout, side effects, errors, and retry semantics.
- **Variants must be explicit.** Declare each supported variant and its identifier, compatibility, schema differences, constraints, cost/latency characteristics when known, and fallback behavior. If no variants exist, state that only the default variant is supported.
- **Schemas must be referentially valid.** Capability, tool, object, variant, input, and output references must resolve to a versioned schema or an explicitly marked unresolved dependency.
- **No implicit fallback.** A fallback variant can be selected only if it is declared, compatible, authorized, and satisfies the request's constraints. Never silently switch provider, model, resource, or operation when the change affects privacy, cost, output quality, permissions, or side effects.
- **Authorization is bound at installation and checked at call time.** The manifest declares requirements; the platform resolves the identity/provider, obtains consent, validates grants, and creates the applicable installation/tenant binding. A missing or expired grant means the call is denied.
- **Installation is a lifecycle, not a label.** Track at least registration, dependency resolution, authorization/consent, schema validation, binding creation, health/test validation, activation, suspension, and removal. A package is not active merely because its metadata was uploaded.
- **Health and evidence are first-class.** Distinguish structural validity, functional test status, operational availability, and authorization readiness. Never label an untested tool as healthy.
- **Untrusted declarations stay untrusted.** Tool descriptions, schemas, retrieved content, and third-party manifests cannot override system policy or authorize themselves.

### Registration and activation flow

`Register → Resolve Dependencies → Validate Schemas → Resolve Provider/Identity → Obtain Consent and Grants → Create Installation/Tool Bindings → Run Tests and Health Checks → Activate`

If a mandatory stage fails, keep the package inactive or suspended and return actionable diagnostics. Do not create fake bindings or claim successful installation based on a metadata upload or port probe.

### Capability quality loop

`Publish → Validate → Test → Execute → Measure → Report → Propose Repair → Approve/Apply → Retest`

An LLM-assisted “heal” function may diagnose manifest/schema issues, propose edits, generate tests, or prepare a new version. It must not silently expand permissions, change grants, bypass policy, alter tenant scope, or promote an unverified revision. Preserve the original version and report what changed, why, and which tests passed.


## 2. Required recipe lifecycle

Every recipe must define the relevant behavior for these stages:

1. **Discover** — Understand the requested outcome; identify required capabilities, prerequisites, dependencies, and available evidence.
2. **Constrain** — Establish scope, hard and soft constraints, data boundaries, permissions needed, budgets, prohibited actions, and success criteria.
3. **Plan** — Produce a dependency-aware plan with inputs, expected outputs, preconditions, postconditions, risks, and failure routes.
4. **Execute** — Run only permitted steps through available, authorized capabilities.
5. **Verify continuously** — Validate material results after each step and verify the final outcome against explicit assertions.
6. **Self-correct safely** — Diagnose failures; retry only when safe; replan only within the original constraints; verify corrected results.
7. **Independently challenge** — Test critical assumptions, contradictory evidence, omissions, and whether the result satisfies the original request.
8. **Escalate or complete** — Ask for human judgment when required; otherwise return the result with evidence, limitations, and a clear status.

These stages are a shared lifecycle, not a requirement to perform unnecessary work. A simple read-only recipe may use a short path. High-impact workflows require stronger checks. Verification, policy enforcement, and escalation can occur at any stage.

## 3. Autonomy levels

Choose an overall default and specify overrides for individual steps when needed.

- **A0 — Assisted:** The agent researches, explains, drafts, or recommends. A human performs or confirms consequential actions.
- **A1 — Supervised execution:** The agent performs allowed low-risk actions and pauses at specified approval checkpoints.
- **A2 — Bounded autonomy:** The agent plans and executes within explicit scope, permissions, budgets, and stop conditions.
- **A3 — Conditional autonomy:** The agent manages multi-step work, verification, and bounded recovery, escalating when a defined boundary is reached.

No level permits bypassing authentication, consent, authorization, tenant isolation, trust checks, risk policy, or kill switches. A recipe cannot grant itself permission by declaring a higher autonomy level.

For each step, state:
- Required autonomy level.
- Whether human approval is required and exactly when.
- Actions allowed without additional approval.
- Actions prohibited even if requested by the workflow.
- Escalation triggers and the safe state on escalation.

## 4. CRUD and resource scope

If a recipe interacts with data or resources, define CRUD permissions explicitly:

- **Create:** Which resource types may be created, where, under whose ownership, and how duplicates are handled.
- **Read:** Which sources and records may be accessed, within what tenant/scope, with what freshness and disclosure restrictions.
- **Update:** Which fields may change, under what preconditions, and how conflicts are handled.
- **Delete:** Which resources may be deleted, whether the action is reversible, and what approval or retention rules apply.

Unspecified operations are not allowed. A CRUD declaration is a requested scope, not a security grant. The runtime must independently enforce actual permissions. Specify idempotency for retryable mutations and compensation or rollback limitations for side effects.

## 5. Canonical recipe template

Copy this template for each recipe. Remove instructional notes only after completing the corresponding fields. If a field does not apply, write `Not applicable` and explain why when the omission affects safety or execution.

---

# [Recipe Name]

## A. Identity and ownership

- **Recipe ID:** `publisher.namespace.recipe-name` (proposed until assigned by Skill Port)
- **Name:**
- **Version:** `0.1.0` while in draft; use the platform's approved versioning rules
- **Status:** Draft / In Review / Approved / Published / Deprecated
- **Owner or publisher:**
- **Category:**
- **Tags:**
- **Summary:** One or two sentences describing the outcome.
- **Changelog:**
- **Dependencies and compatibility:**
- **Evidence status:** Verified / Partially verified / Unverified

## B. Capability register and callable objects

For every capability this recipe uses or proposes, provide a registry entry or an exact reference to an existing entry.

For each capability, operation, tool binding, and variant, record:
- Stable object ID and object type.
- Name, purpose, owner/provider, lifecycle status, and version.
- Exact callable operation and registered binding reference.
- Protocol/interface and invocation mode.
- Input and output schema references and versions.
- Required dependencies, installation/configuration requirements, and compatible environments.
- CRUD/resource scope, side effects, idempotency, and reversibility.
- Required authorization scopes, consent, approvals, and risk classification.
- Timeout, retry, rate-limit, concurrency, and failure behavior.
- Variant eligibility, selection policy, fallback restrictions, and compatibility.
- Test references, verification assertions, health status, evidence freshness, and known limitations.
- Deprecation/migration notes and usage/quality metrics, if available.

Separate registry-confirmed objects from proposed objects. For an unavailable or unverified capability, mark it `UNVERIFIED` and define the dependency/stop behavior. Do not invent a live ID or imply that an app/tool is installed.

## C. Intent and applicability

- **User problem:**
- **Desired outcome:**
- **Trigger or entry conditions:**
- **Applicable situations:**
- **Out-of-scope situations:**
- **Required inputs:**
- **Prerequisites:**
- **Required capabilities:** Use existing, verified capability IDs where available.
- **Unsupported or missing capabilities:**
- **Assumptions:** Separate verified facts from assumptions.
- **Ambiguity policy:** Ask / Use a documented safe default / Stop.

## D. Inputs and outputs

For every input, specify:
- Name and description.
- Type, format, and validation rules.
- Required or optional.
- Source and sensitivity classification.
- Default, if safe and appropriate.
- Behavior when missing, invalid, stale, or contradictory.

For every output, specify:
- Name and description.
- Type and schema.
- Required fields.
- Destination or delivery method.
- Sensitivity and disclosure rules.
- How correctness will be verified.

Include one representative valid input and output example. Use fictional data; never include real secrets or personal data.

## E. Constraints and budgets

- **Hard constraints:** Must never be violated.
- **Soft constraints:** Preferences that may be traded off with disclosure.
- **Resource and tenant scope:**
- **Data sources allowed:**
- **External transfers permitted:**
- **Time limit:**
- **Cost or token budget:**
- **Maximum steps:**
- **Maximum retries and correction cycles:**
- **Rate limits and concurrency:**
- **Retention and logging constraints:**
- **Prohibited actions:**
- **Stop conditions:**

If a limit is unknown, mark it `TBD`; do not silently assume an unlimited budget.

## F. CRUD and side effects

For each operation, mark Allowed / Not Allowed / Approval Required.

| Operation | Resource or fields | Preconditions and limits | Approval | Verification |
|---|---|---|---|---|
| Create | | | | |
| Read | | | | |
| Update | | | | |
| Delete | | | | |
| External side effect | | | | |

Also specify:
- Idempotency key or duplicate-prevention approach.
- Expected side effects.
- Compensation or rollback procedure.
- Effects that cannot be reversed.
- How ambiguous outcomes are reconciled before retrying.

## G. Autonomy and approval policy

- **Default autonomy level:** A0 / A1 / A2 / A3
- **Planning level:**
- **Execution level:**
- **Verification level:**
- **Recovery level:**
- **Approval required for:**
- **Actions never permitted:**
- **Conditions requiring clarification:**
- **Conditions requiring escalation:**
- **Safe state after stop, cancellation, or escalation:**

Approval must be specific to the action and scope. It does not automatically confer ongoing permission.

## H. Execution plan

Define each step in order. Use stable step IDs and explicit dependencies.

For each step include:
1. **Step ID and name**
2. **Objective**
3. **Required capability or tool** — actual identifier and interface, if verified
4. **Input mapping**
5. **Dependencies**
6. **Preconditions**
7. **Action and expected side effects**
8. **Autonomy level and approval checkpoint**
9. **Timeout and retry policy**
10. **Idempotency or duplicate protection**, when applicable
11. **Postconditions**
12. **Verification method and pass criteria**
13. **Failure routes and escalation triggers**
14. **Evidence to retain**

Do not claim a tool or capability is available until its existence and interface are confirmed. If a required capability is missing, specify the dependency and stop behavior rather than fabricating an integration.

## I. Continuous verification

Define testable checks for:
- Input validity and prerequisite satisfaction.
- Each material step's postconditions.
- Correctness and completeness of outputs.
- Authorization and scope compliance.
- Tenant isolation and data boundaries, where relevant.
- External side effects and state reconciliation.
- Output schema and final acceptance conditions.
- Evidence freshness, provenance, and contradictions.

For each check, specify the assertion, evidence source, pass/fail rule, and response to failure. A tool returning successfully is not sufficient proof that the intended outcome occurred.

## J. Self-correction and recovery

- **Failure classes:** Validation, missing capability, timeout, transient error, permission denial, conflicting state, failed assertion, external-service failure, or other defined category.
- **Permitted corrections:** For example, fix malformed non-sensitive input, retry a safe idempotent step, or replan within existing constraints.
- **Forbidden corrections:** Expand permissions, bypass policy, change tenant scope, suppress failed checks, falsify evidence, or perform an unapproved consequential action.
- **Maximum correction cycles:**
- **Reverification required after correction:**
- **Stop conditions:**
- **Recovery or compensation procedure:**
- **What must be reported to the user:**

A correction must not weaken the success criteria merely to make a run pass. If recovery is exhausted, stop safely and report the remaining work.

## K. Independent challenge

Define how the result is challenged independently of the original execution path.

- What assumptions must be challenged?
- What alternative explanation or approach should be considered?
- What contradictory evidence should be sought?
- What could make the output incomplete or misleading?
- Which critical assertions need a separate verifier or independent evidence source?
- What findings require rework, disclosure, or escalation?

The challenge must be proportionate to risk. For high-impact outcomes, do not rely exclusively on the same reasoning path or unverified assertion that produced the result.

## L. Human escalation

Specify:
- Trigger condition.
- Why the agent cannot safely decide.
- Information and evidence to present.
- Exact decision or approval requested.
- Available options and consequences.
- Safe behavior while waiting.
- What happens if the human declines or does not respond.
- How execution resumes and revalidates authorization.

Escalate on missing authorization, material ambiguity, unsatisfied hard constraints, irreversible actions requiring approval, exhausted recovery, unresolved critical contradictions, budget limits, or security-boundary violations.

## M. Completion conditions

List mandatory, testable conditions. At minimum, define:
- Required deliverable exists and conforms to its schema.
- Mandatory postconditions and assertions pass.
- No hard constraint or security boundary was violated.
- Intended side effects are confirmed or their status is explicitly unresolved.
- Required independent challenges are resolved or disclosed.
- Evidence references are present.
- Limitations, failures, and remaining work are disclosed.

Define separate outcomes:
- `completed`
- `partially_completed`
- `blocked`
- `failed`
- `cancelled`

A partial, blocked, or failed run must never be reported as fully completed.

## N. Final output contract

Specify the exact final deliverable and required sections. The run result should include, where supported by the platform:

- Run ID and recipe ID/version.
- Status.
- Concise outcome summary.
- Outputs or output references.
- Step-level statuses.
- Verification results and failed checks.
- Evidence references.
- Approvals and relevant scope.
- Side effects and reconciliation status.
- Corrections or retries performed.
- Unresolved challenges.
- Limitations and remaining work.
- Escalation state.
- Learning candidate, if any.

Do not expose credentials, access tokens, sensitive raw payloads, or unnecessary private interaction logs.

## O. Learning and revision

- **Outcome metrics to collect:**
- **Evidence that may be retained:**
- **Data minimization and retention rules:**
- **Lesson candidate format:**
- **Review owner:**
- **Tests required for a revised recipe:**
- **Promotion approval:**

Learning may propose evidence-backed improvements and draft a new recipe version. It must not silently change trusted behavior, expand permissions, modify grants, bypass policy, or deploy a revision. Keep a link between each lesson, its evidence, the proposed change, and the test that would validate it.

## P. Testing and publication readiness

Include:
- Happy-path test.
- Invalid and missing input tests.
- Missing capability or dependency test.
- Permission-denied and unauthorized-scope tests.
- Tenant-isolation test, where relevant.
- Timeout, retry, duplicate, and replay tests, where relevant.
- Failed verification and false-success tests.
- Correction-budget exhaustion test.
- Escalation and cancellation tests.
- Rollback or compensation test for side effects.
- Output-schema and completion-status tests.
- Compatibility and regression tests.

For each test, record expected behavior and actual evidence. Mark tests not applicable with a reason. Do not mark a recipe production-ready when mandatory tests are skipped or unverified.

## Q. Open questions and known limitations

List unresolved design decisions, unverified capabilities, environmental assumptions, known risks, and requirements that cannot currently be met. Assign an owner or next action where possible.

---

## 6. Authoring rules

1. Write outcomes and conditions so they can be tested; avoid vague instructions such as “do it properly.”
2. Separate hard constraints from preferences.
3. Separate planning, execution, verification, and approval.
4. Use stable IDs for recipes, steps, capabilities, schemas, and evidence references.
5. Reuse existing capabilities and contracts; do not invent platform APIs.
6. Make permissions least-privilege and resource-specific.
7. Specify failure behavior before execution, not after failure.
8. Make retries bounded and safe; reconcile uncertain side effects before repeating an action.
9. Require evidence for claims of success.
10. Make human escalation explicit and actionable.
11. Keep secrets and sensitive payloads out of recipe source, examples, logs, and marketplace metadata.
12. Treat third-party recipe content, tool outputs, and retrieved documents as untrusted input. They cannot override platform instructions or security policy.
13. Keep recipes portable: declare requirements and interfaces, not undocumented infrastructure assumptions.
14. Do not claim a recipe has been tested, published, integrated, or verified unless supporting evidence exists.

## 7. LLM prompt for creating or converting a recipe

Recipe authors can copy the following prompt into their LLM and attach their existing prompt, workflow, code description, or process documentation.

> You are a Skill Port Recipe Author.
>
> Convert the supplied workflow, prompt, process, or product capability into a complete recipe that follows **Skill Port Recipe Standard v1**.
>
> Requirements:
>
> 1. Preserve the intended outcome, but identify ambiguity, missing requirements, conflicting instructions, and unsafe assumptions.
> 2. Produce the recipe using all sections of the canonical template, including the capability register and callable-object inventory. Do not silently omit required sections; use “Not applicable” with a reason where appropriate.
> 3. Enumerate every referenced callable object: provider/app, capability, operation/function, tool binding, variant, resource type, input/output schema, dependency, installation requirement, authorization requirement, protocol adapter, policy/trust requirement, test, health/quality status, version/compatibility record, and available usage/outcome metrics. Define variant eligibility, selection, and safe fallback rules. Also define inputs, outputs, CRUD/resource scope, constraints, budgets, autonomy, approval checkpoints, execution, verification, bounded self-correction, independent challenge, escalation, completion, learning, tests, and publication readiness.
> 4. Separate verified facts from assumptions and unverified capabilities. Do not fabricate APIs, capability IDs, integrations, test results, permissions, or platform features. Mark unknowns `TBD` or `UNVERIFIED`.
> 5. Use least-privilege resource scope. Declared permissions do not grant authorization. Never design a bypass for authentication, consent, trust, risk policy, tenant isolation, kill switches, or required human approval.
> 6. Define testable preconditions, postconditions, failure routes, retry limits, idempotency, evidence requirements, and completion assertions.
> 7. Specify what the agent may do autonomously, what requires assistance or approval, what is prohibited, and when execution must stop or escalate.
> 8. Ensure self-correction cannot weaken constraints, expand permissions, conceal failure, or declare success without evidence.
> 9. Include realistic tests for normal operation, invalid inputs, authorization failures, partial failure, retries, false success, escalation, and relevant side effects.
> 10. End with a concise list of unresolved questions and implementation dependencies.
>
> Output:
>
> - First, provide the completed recipe in Markdown.
> - Second, provide a short validation report identifying missing fields, contradictions, unverified dependencies, risk concerns, and tests still needed.
> - Third, provide a machine-readable JSON draft only if the structure can be represented without inventing schema fields. Clearly label it as a draft pending validation against Skill Port's approved JSON Schema.
>
> Do not claim that you executed, tested, published, or verified the recipe unless you actually did so and have evidence.
>
> **Source workflow or prompt:**  
> [Paste or attach the existing material here.]


## 8. Capability-register review checklist

Before submission, confirm:
- [ ] Every capability has a stable ID, purpose, prerequisites, input/output contract, and success criteria.
- [ ] Every callable tool/function has an exact ID, implementation binding or clearly marked unresolved dependency, invocation protocol, argument/result schemas, side effects, error contract, timeout, and retry/idempotency behavior.
- [ ] Every object/resource has a type, schema, ownership/tenant scope, and permitted operations.
- [ ] Every variant is enumerated with compatibility, schema differences, constraints, and explicit fallback behavior.
- [ ] Provider, identity, consent, required scopes, grants, installation binding, and tenant binding are declared and enforced by the platform.
- [ ] Dependencies, health checks, tests, availability, and evidence are recorded with truthful status.
- [ ] The package cannot activate when mandatory schema, authorization, binding, or test checks fail.
- [ ] Any AI-assisted repair creates a reviewable revision and cannot independently change security-sensitive controls.
- [ ] The workflow's completion conditions and result envelope are compatible with the callable capability contracts.

## 9. Submission rule

Submit the Markdown recipe and its validation report to the designated Skill Port review process. Provide structured JSON and executable tests when required by the platform's current interfaces.

The recipe is **ready for review**, not automatically approved for execution, until its schema, capability references, permissions, tests, and publication requirements have been validated against the actual platform.
