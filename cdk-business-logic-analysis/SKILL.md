---
name: "cdk-business-logic-analysis"
description: "Defines how to analyze functional requirements and an existing CDK repository to identify business rules, conditions, validations, transformations, orchestration, logic placement, error handling, and retry behavior."
version: 1
---

## When to Use

Use when analyzing business behavior that must be implemented as part of new or extended CDK infrastructure in an existing repository.

## Objective

Create a `BusinessLogicDesign` that clearly identifies business rules, processing behavior, decision logic, and the appropriate location for each business responsibility.

## Inputs

- `RequirementsModel`
- `RepoProfile`

## Analysis

Analyze the requirements and repository profile to identify:

1. Business rules.
2. Conditions and decision points.
3. Validation requirements.
4. Data transformations.
5. Processing logic.
6. Orchestration requirements.
7. Error handling.
8. Retry behavior.
9. Idempotency requirements.
10. Existing components that already implement relevant logic.
11. The appropriate location for each business responsibility.

## Business Rule Analysis

Extract each explicit business rule from the requirements.

For each rule, identify:

- Rule.
- Trigger.
- Inputs.
- Conditions.
- Expected behavior.
- Output or outcome.
- Component responsible for execution.

Do not invent business rules.

## Conditional Logic

Identify:

- Conditional branches.
- Decision points.
- Routing conditions.
- Event filtering conditions.
- Validation conditions.
- Success and failure paths.

Represent complex logic as explicit decision flows where useful.

## Validation

Identify validation requirements including:

- Input validation.
- Request validation.
- Business validation.
- Data validation.
- Event validation.
- External service response validation.

Identify the appropriate component responsible for each validation.

## Data Transformation

Identify:

- Input transformations.
- Request mapping.
- Response mapping.
- Event transformation.
- Data enrichment.
- Data normalization.

Identify where each transformation should occur.

## Business Logic Placement

Determine the most appropriate location for each business responsibility.

Possible locations include:

- API Gateway
- Lambda
- EventBridge
- SQS
- Step Functions
- DynamoDB
- Existing application services
- Existing constructs or components
- External services

Do not assume all business logic belongs in Lambda.

Prefer existing application components when they already provide the required behavior.

## Orchestration

Identify workflows that require:

- Sequential processing.
- Parallel processing.
- Conditional branching.
- Event-driven processing.
- Asynchronous processing.
- Long-running workflows.

Determine whether orchestration should be handled by an existing component or a new workflow mechanism.

Do not prescribe a specific AWS service unless supported by the requirements or repository architecture.

## Error Handling

Identify:

- Expected business errors.
- Validation failures.
- Integration failures.
- Processing failures.
- Retryable failures.
- Non-retryable failures.
- Dead-letter or failure handling requirements.

Clearly distinguish business errors from infrastructure or integration failures.

## Retry and Resilience

Identify where retry behavior is required.

For each retry scenario, identify:

- Operation.
- Failure condition.
- Retryable or non-retryable.
- Retry responsibility.
- Maximum retry behavior, if specified.
- Failure outcome.

Do not invent retry counts or policies when not provided.

## Idempotency

Identify operations where duplicate processing may cause incorrect business outcomes.

Determine:

- Whether idempotency is required.
- Idempotency key or identifier, if defined.
- Component responsible for enforcing idempotency.

If information is unavailable, record a gap.

## Existing Logic Reuse

Use `RepoProfile` to identify existing:

- Business logic components.
- Validation utilities.
- Transformation utilities.
- Shared constructs.
- Processing patterns.
- Error handling patterns.

Prefer reuse or extension over duplicating existing behavior.

## Guardrails

- Do not generate CDK code.
- Do not modify the repository.
- Do not invent business rules.
- Do not invent missing conditions.
- Do not assume business logic belongs in Lambda.
- Do not duplicate existing business logic unnecessarily.
- Do not make resource design decisions outside the business logic analysis.
- Do not prescribe an AWS service without sufficient evidence.
- Distinguish business logic from infrastructure configuration.
- Distinguish business errors from technical failures.
- Clearly identify assumptions.
- Clearly identify gaps.

## Output

Produce a structured `BusinessLogicDesign` containing:

### Business Rules

For each rule:

- Rule.
- Trigger.
- Inputs.
- Conditions.
- Expected behavior.
- Outcome.
- Responsible component.

### Decision Logic

- Conditions.
- Decision points.
- Branches.
- Success paths.
- Failure paths.

### Validation

- Validation type.
- Validation rule.
- Responsible component.

### Transformations

- Input.
- Transformation.
- Output.
- Responsible component.

### Logic Placement

For each business responsibility:

- Business responsibility.
- Recommended component.
- Reason.
- Existing component to reuse, if applicable.

### Orchestration

- Workflow.
- Processing sequence.
- Parallel processing.
- Conditional branching.
- Asynchronous behavior.

### Error Handling

- Error scenario.
- Error type.
- Handling behavior.
- Responsible component.

### Retry and Resilience

- Operation.
- Failure condition.
- Retryability.
- Retry responsibility.
- Failure outcome.

### Idempotency

- Operation.
- Idempotency requirement.
- Identifier or key.
- Responsible component.

### Existing Logic Reuse

List existing repository components or patterns that should be reused or extended.

### Assumptions

List assumptions explicitly.

### Gaps

List unresolved business logic information.

## Verification

Before returning `BusinessLogicDesign`, verify:

1. All explicit business rules are captured.
2. Conditions and decision points are identified.
3. Validation requirements are mapped.
4. Data transformations are identified.
5. Business logic placement is defined.
6. Existing logic was evaluated for reuse.
7. Orchestration requirements are identified.
8. Error handling is defined.
9. Retry behavior is identified where applicable.
10. Idempotency requirements are considered.
11. Business errors are distinguished from technical failures.
12. Assumptions are clearly documented.
13. Gaps are clearly documented.
14. No CDK code is generated.

## Scope Discipline

`additionalInformation` from the Input Manifest is the authoritative scope filter for the
run, and it reaches this stage through `RequirementsModel.scope`.

- Cover only what `scope.inScope` contains. Record anything else as deferred or
  out-of-scope; never silently include it because it shares a stack or feature.
- If `scope.authoritativeScopeFilter` is empty, blank, or a generic non-restrictive
  placeholder — `NA`, an empty value, or wording that names no specific resource or
  feature restriction — there is NO restriction: cover
  everything the RequirementsModel identifies as in scope rather than narrowing further.
- Copy literal names, identifiers and ARN patterns from the source documents verbatim.
  Never invent one; anything not given becomes a declared gap.
- Report gaps as structured objects with `id`, `field`, `description`,
  `requiresHumanInput` and `blocksCodeGeneration` — never as free-text strings.

## Output Location

Write the result as valid JSON (no markdown or prose wrapper) to `workflow_output/CDKBusinessLogicAnalysisAgent.json`.

`workflow_output` lives at the workflow RUN ROOT: the directory that CONTAINS the cloned
repository's `src/` folder. It must never be created inside `src/`. The working directory
may already be `.../src`, so resolve it first rather than using a bare relative path:

```text
ROOT="$(pwd)"; case "$ROOT" in */src) ROOT="$(dirname "$ROOT")";; esac
mkdir -p "$ROOT/workflow_output"
```

Write to `$ROOT/workflow_output/CDKBusinessLogicAnalysisAgent.json` and, if reporting the location back, report the
full absolute path. Never emit an unsubstituted placeholder such as `<ROOT>`. When reading
a file from this folder, try `workflow_output/<file>` and fall back to
`../workflow_output/<file>`; ignore any stale copy under `src/workflow_output/`.

Writing the file is a side effect. Your final answer text must literally BE the complete
`BusinessLogicDesign` JSON object — never a summary, a narrative, or an acknowledgement such as
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
