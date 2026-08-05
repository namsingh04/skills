---
name: "cdk-requirements-analysis"
description: "Guidelines for analyzing Solution Design and Acceptance Criteria to produce a structured, traceable Requirements Model for reusable CDK infrastructure generation. Identifies functional requirements, infrastructure capabilities, resource relationships, business logic, integrations, dependencies, and gaps without making repository-specific implementation decisions."
version: 1
created: "2026-07-27"
updated: "2026-07-27"
---

## When to Use

Use this skill in the requirements analysis stage of a reusable CDK infrastructure generation workflow.

Apply when the workflow has:

- A Solution Design document
- Jira template or Acceptance Criteria
- An Input Manifest produced by the InputArtifactRegistrationAgent

The output of this skill is consumed by downstream infrastructure design and specification agents.

This skill analyzes **what the solution must do**.

It does not determine **how the existing CDK repository should be changed**.

---

## Inputs

### Required

1. **Solution Design**
   - Describes the proposed solution, architecture, flows, and intended behavior.

2. **Acceptance Criteria**
   - Defines expected functional and behavioral outcomes.

### Context

3. **Input Manifest**
   - Provides references to registered artifacts.
   - Provides traceability information.

Do not require or deeply analyze `standard.md` during this stage.

`standard.md` must be preserved and passed to the downstream CDK code generation stage.

---

## Primary Objective

Transform the Solution Design and Acceptance Criteria into a structured and traceable `RequirementsModel`.

The Requirements Model must provide downstream agents with a complete understanding of:

- What the system must do
- What infrastructure capabilities are required
- How resources interact
- Where business logic exists
- How external systems are integrated
- What dependencies exist
- What information is missing or ambiguous

---

## Procedure

### 1. Analyze Solution Design

Extract all relevant requirements and architectural intent from the Solution Design.

Identify:

- System capabilities
- Functional behavior
- Application flows
- Event flows
- Data flows
- Infrastructure requirements
- Resource responsibilities
- External integrations
- Business rules
- Conditions and decision logic
- Dependencies
- Configuration requirements
- Security requirements explicitly stated in the document

Do not make repository-specific implementation decisions.

---

### 2. Analyze Acceptance Criteria

Extract expected behaviors and outcomes from the Acceptance Criteria.

For each criterion:

- Identify the expected behavior.
- Identify the relevant requirement.
- Identify related infrastructure or integration needs.
- Preserve traceability to the source.

Where possible, map acceptance criteria to requirement IDs.

Do not assume an acceptance criterion is satisfied unless the source artifacts explicitly support that conclusion.

---

### 3. Extract Functional Requirements

Identify requirements describing what the system must do.

Examples include:

- Receive an event.
- Validate a request.
- Route a message.
- Transform data.
- Persist data.
- Publish an event.
- Consume a queue message.
- Call an external service.
- Apply business rules.
- Retry failed processing.
- Handle conditional flows.

Assign a unique requirement ID to each extracted requirement.

Use the format:

```text
REQ-001
REQ-002
REQ-003

```

---

### 4. Extract Resource Requirements

This is the most important section of the model. Every downstream stage — resource design,
the infrastructure specification, and the generated code — reads physical resource names
from here and from nowhere else. When it is missing, names surface downstream as unfilled
configuration fields and the generated code is unusable.

Produce **one entry per row** of every table in the Solution Design that lists stacks or
resources — whatever those tables are titled — plus any resource named elsewhere in the
document. Use the `structured-table-extraction` skill's row objects rather than re-reading
the table as prose.

Each entry carries:

- `resourceType` — the CloudFormation type where the source states or implies one;
- `stackName` — that row's stack cell, copied character for character;
- `namePattern` — that row's resource name, copied character for character;
- `requirements` — the requirement ids this resource satisfies;
- `notes` — one string per value in that row's configuration cell, losing nothing.

Include **every** row, including rows whose resource type falls outside the current run's
scope. Scope filtering happens downstream; the job here is to lose nothing, because a name
dropped here cannot be recovered by any later stage.

Names are verbatim: never abbreviate, expand, re-case, or apply a naming convention of your
own. Templated placeholder tokens are reproduced byte for byte — never resolved, never
substituted for an environment, never allowed to become the string `null`, and never a
reason to drop a row. Any name beginning `null-` is a bug.

Also capture the AWS capabilities the solution requires — messaging, routing, persistence,
compute, storage, notification, security, observability — with the purpose each serves and
the requirement ids it supports.

---

### 5. Compute the Scope Directive

`additionalInformation` from the Input Manifest is an **instruction** supplied by the
requester. It has two possible readings, and you decide which applies once, here, so every
downstream stage inherits the same answer.

Copy the raw value verbatim into `scope.authoritativeScopeFilter` either way. Never null,
never paraphrased.

**Non-restrictive.** The value is `NA`, empty, blank, or wording that names no specific
resource, feature or restriction. Then it carries no instruction and imposes no
restriction: it is not considered further. Everything the Solution Design and Acceptance
Criteria describe is in scope. Populate `scope.inScope` with all of it, leave
`scope.deferredScope` empty, and leave `scope.mandatoryInstructions` empty.

**An instruction.** The value says something specific. Then it is authoritative and what it
asks for is **mandatorily in scope** — record each distinct ask as an entry in
`scope.mandatoryInstructions`, verbatim. Where it also names resource types, it narrows
scope as well: map those to concrete AWS resource types, include only those types'
requirements in `scope.inScope`, and put everything else the Solution Design describes into
`scope.deferredScope` with a one-line reason.

Belonging to the same Solution Design stack or feature does NOT put a resource type in
scope when the instruction restricts scope. Never merge, drop, or silently reclassify a
deferred item.

---

### 6. Identify Resource Relationships, Business Logic, Integrations and Dependencies

Capture how resources interact, where business rules live, which external systems are
involved, and what this work depends on. Preserve the direction of every flow.

---

### 7. Identify Gaps

Record every value the downstream stages need that the source artifacts do not supply.

Report each gap as a structured object, never as free text.

---

## Guardrails

- Do not design the CDK implementation.
- Do not make repository-specific decisions.
- Do not generate CDK code.
- Do not invent a name, identifier, ARN, threshold, or configuration value. Copy literal
  values from the Solution Design verbatim; anything not given becomes a gap.
- Do not include a resource type that `additionalInformation` did not request.
- Do not narrate your process or summarise what you read. Return the model itself.
- Clearly identify assumptions.
- Clearly identify gaps.

## Output

Produce a structured `RequirementsModel`. The top level must match this shape exactly.
Use these key names and this nesting literally — do not rename or relocate them, and do
not place `inScope` or `deferredScope` at the document root:

```json
{
  "modelType": "RequirementsModel",
  "scope": {
    "authoritativeScopeFilter": "<additionalInformation copied verbatim>",
    "mandatoryInstructions": ["<each distinct ask, verbatim; empty when non-restrictive>"],
    "inScope": ["<resource type or capability>"],
    "deferredScope": [{ "item": "<name>", "reason": "<one line>" }]
  },
  "sourceArtifacts": {
    "solutionDesign": { "status": "available", "reference": "<ref>" },
    "acceptanceCriteria": { "status": "available", "reference": "<ref>" }
  },
  "functionalRequirements": [
    {
      "id": "REQ-001",
      "statement": "<what the system must do>",
      "category": "<functional|resource|configuration>",
      "sourceRefs": ["<document section>"],
      "acceptanceCriteriaRefs": ["AC01"]
    }
  ],
  "resourceRequirements": [
    {
      "resourceType": "<CloudFormation type>",
      "stackName": "<verbatim stack cell>",
      "namePattern": "<verbatim resource name>",
      "requirements": ["REQ-001"],
      "notes": ["<every value from the configuration cell, one per string>"],
      "sourceRefs": ["<table caption or heading>"]
    }
  ],
  "infrastructureRequirements": [
    {
      "id": "INF-001",
      "resourceType": "<AWS service>",
      "literalName": "<verbatim name from the Solution Design, or an empty string>",
      "configuration": { },
      "sourceRefs": ["<document section>"]
    }
  ],
  "resourceRelationships": [
    { "source": "<resource>", "target": "<resource>", "interaction": "<verb>" }
  ],
  "businessLogic": [],
  "integrations": [],
  "dependencies": [],
  "acceptanceCriteriaTraceability": [
    { "criterion": "AC01", "requirementIds": ["REQ-001"], "covered": true }
  ],
  "gaps": [
    {
      "id": "GAP-001",
      "field": "<the missing field>",
      "description": "<what is missing and why it matters>",
      "requiresHumanInput": true,
      "blocksCodeGeneration": false
    }
  ]
}
```

Your final answer text must literally BE this JSON object. Writing it to an output file
is a side effect; it never replaces returning the full model as your answer.

Every ARN pattern, code expression, or string concatenation must be written as one
properly escaped JSON string, never as bare unquoted code spliced into array or object
syntax. Never splice markdown tables, pipe-delimited rows, or nested backticks into a
JSON string without escaping them — describe such content in plain prose instead.

## Verification

Before returning `RequirementsModel`, verify:

1. A top-level key named `scope` exists.
2. `authoritativeScopeFilter` is inside `scope` — not a top-level `scopeFilter`, not
   `authoritativeFilter`, not `generatedFrom.scopeFilter`.
3. `inScope` is inside `scope`, not at the document root.
4. `deferredScope` is inside `scope`, not at the document root.
5. `authoritativeScopeFilter` matches `additionalInformation` verbatim.
5a. `resourceRequirements` is present and has one entry per row of every stack/resource
    table in the Solution Design, including rows outside the current scope.
5b. Every `stackName` and `namePattern` matches its source cell character for character,
    every templated placeholder token is intact, and no value is `null` or begins `null-`.
6. Every requirement has a unique `REQ-###` id and at least one source reference.
7. Every acceptance criterion is mapped or explicitly recorded as uncovered.
8. Every literal name is copied verbatim from a source document.
9. Every gap is a structured object, not a string.
10. The whole answer parses as valid JSON with balanced braces and brackets.

Items 1 through 4 must all hold simultaneously. A model that satisfies some and not
others is rejected exactly like one that satisfies none.

## Output Location

Write the result as valid JSON (no markdown or prose wrapper) to `workflow_output/CDKRequirementsAnalysisAgent.json`.

`workflow_output` lives at the workflow RUN ROOT: the directory that CONTAINS the cloned
repository's `src/` folder. It must never be created inside `src/`. The working directory
may already be `.../src`, so resolve it first rather than using a bare relative path:

```text
ROOT="$(pwd)"; case "$ROOT" in */src) ROOT="$(dirname "$ROOT")";; esac
mkdir -p "$ROOT/workflow_output"
```

Write to `$ROOT/workflow_output/CDKRequirementsAnalysisAgent.json` and, if reporting the location back, report the
full absolute path. Never emit an unsubstituted placeholder such as `<ROOT>`. When reading
a file from this folder, try `workflow_output/<file>` and fall back to
`../workflow_output/<file>`; ignore any stale copy under `src/workflow_output/`.

Writing the file is a side effect. Your final answer text must literally BE the complete
`RequirementsModel` JSON object — never a summary, a narrative, or an acknowledgement such as
"generated successfully".

## JSON Output Contract

- The answer must parse as valid JSON: balanced braces and brackets, no trailing keys
  after the outer object has closed, no markdown fence around it.
- Every ARN pattern, code expression, or string concatenation must be one properly escaped
  JSON string — never bare unquoted code spliced into array or object syntax.
- Never splice markdown tables, pipe-delimited rows, or nested backticks into a JSON string
  without escaping them; describe such content in plain prose instead.
- Never bake a literal `null` into a resource name or ARN where a real value or a declared
  gap belongs.
- On a retry after validation feedback, always return the complete corrected model, never
  only the changed fields and never a confirmation message.

## Status Contract

This skill's model is emitted inside the shared workflow envelope defined by the
`workflow-status-contract` skill. Alongside this model's own top-level keys — as siblings,
never as a wrapper around them — every output carries `status`, `stage`, `runId`,
`outputPath`, `upstreamStatus`, `nextAction`, `gaps` and `warnings`.

Read the upstream `status` before doing anything else:

- `OK` or `PARTIAL` — proceed. `PARTIAL` means work with the gaps you were given; it is
  never a reason to stop.
- `BLOCKED` or `SKIPPED` — do not fail and do not raise. Write your own output file with
  `status: "SKIPPED"`, the `upstreamStatus` you saw, an empty payload and
  `nextAction: "SKIP_DOWNSTREAM"`, then return.
- Upstream missing or unreadable — **fail open**. Proceed as if it were `OK` and record a
  warning. Inability to see an upstream result is never grounds to block.

`BLOCKED` is reserved for this stage's own unrecoverable failure: its input is missing,
empty or unparseable, or its own tool calls failed beyond retry. A gap count, a severity
judgement, or a downstream readiness flag never produces `BLOCKED`.

Every gap is an object carrying exactly `id`, `field`, `description`, `source`,
`requiresHumanInput`, `blocksCodeGeneration`, `suggestedResolution` and `resolution`.
`resolution` is an empty string when this stage creates the gap — only a human review gate
fills it in.

## Authority Chain

Resolve every "where does this value come from?" question in this order:

1. **The standards file** — the base. Conventions, policy, naming, structure, required
   commands and required checks.
2. **The Solution Design** — what is being built: resource names, keys, settings, flows,
   and any environment or account values it states.
3. **The RepoProfile** — the repository as it actually is.
4. **A declared gap** — when no source states the value.

Never invent a value. When two sources disagree, take the higher-ranked one **and** record
a `warnings` entry naming both — never resolve a conflict silently. One exception: for
mechanical facts required to compile — the symbol names, helper signatures, file paths and
import specifiers that actually exist in the checkout — the RepoProfile wins, because the
standards file describes policy and the compiler does not negotiate. Record the conflict
either way.

A value stated by any source is never a gap. Check the sources before writing one: listing
the same value as both a requirement and a gap is a self-contradiction.
