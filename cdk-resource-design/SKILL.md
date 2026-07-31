---
name: "cdk-resource-design"
description: "Defines how to analyze CDK requirements and an existing CDK repository to determine resource reuse, extension, creation, references, stack and construct placement, dependencies, relationships, and IAM strategy."
version: 1
---

## When to Use

Use when designing AWS CDK infrastructure that will be added to an existing CDK repository.

## Objective

Create a `ResourceDesign` that defines the required AWS resources and how they fit into the existing CDK repository.

## Inputs

- `RequirementsModel`
- `RepoProfile`

## Analysis

Analyze the requirements and repository profile to determine:

1. Required AWS resources.
2. Existing resources that can satisfy requirements.
3. Existing stacks and constructs that can be extended.
4. New resources that must be created.
5. Resources that should be referenced from other stacks or shared infrastructure.
6. Resource configuration requirements.
7. Resource dependencies.
8. Resource-to-resource relationships.
9. Existing IAM roles that can be reused.
10. Required permissions that are not already available.
11. Repository patterns and conventions that must be followed.

## Resource Classification

Classify every relevant resource as exactly one of:

- `REUSE` — use an existing resource without creating a new one.
- `EXTEND` — add functionality to an existing stack or construct.
- `CREATE` — create a new resource.
- `REFERENCE` — consume or reference a resource managed elsewhere.

Provide a clear reason for each classification.

## Stack and Construct Placement

Determine:

- Target stack.
- Target construct.
- Existing construct to extend, if applicable.
- New construct required, if applicable.
- Files likely to be modified.
- Files likely to be created.

Prefer existing repository patterns over introducing new architectural patterns.

## Resource Relationships

Explicitly describe relationships such as:

- API Gateway → Lambda
- Lambda → EventBridge
- EventBridge → SQS
- SQS → Lambda
- Lambda → DynamoDB
- Resource → External Service

Do not assume relationships that are not supported by the requirements or repository profile.

For each relationship, identify:

- Source resource.
- Target resource.
- Interaction type.
- Dependency.
- Required permissions, if applicable.

## IAM Strategy

For each resource requiring permissions:

1. Identify existing IAM roles that can be reused.
2. Identify existing policies or permissions that can be reused.
3. Identify additional permissions required.
4. Recommend new IAM roles only when existing roles cannot satisfy the requirement.
5. Never duplicate an existing IAM role without justification.

## Reuse Strategy

Prioritize:

1. Existing resources.
2. Existing stacks.
3. Existing constructs.
4. Existing IAM roles.
5. Existing repository patterns.

Only recommend new infrastructure when reuse or extension is insufficient.

## Guardrails

- Do not generate CDK code.
- Do not modify the repository.
- Do not invent resources that are not supported by the requirements.
- Do not invent repository components that are not present in `RepoProfile`.
- Do not create duplicate resources unnecessarily.
- Do not create duplicate IAM roles unnecessarily.
- Do not assume all resources belong in a new stack.
- Do not assume business logic belongs in Lambda.
- Do not make decisions outside resource architecture and design.
- Clearly identify assumptions.
- Clearly identify gaps or missing information.

## Output

Produce a structured `ResourceDesign` containing:

### Repository Integration

- Target stack.
- Target construct.
- Existing constructs to extend.
- Files to modify.
- Files to create.

### Resource Plan

For each resource:

- Resource name.
- AWS service/type.
- Classification: `REUSE`, `EXTEND`, `CREATE`, or `REFERENCE`.
- Purpose.
- Configuration.
- Reason for classification.

### Resource Relationships

For each relationship:

- Source.
- Target.
- Interaction.
- Dependency.
- Required permissions.

### IAM Strategy

- Existing IAM roles to reuse.
- Existing policies to reuse.
- Additional permissions.
- New IAM roles required, if any.
- Justification for new IAM resources.

### Dependencies

- Resource dependencies.
- Stack dependencies.
- Construct dependencies.

### Assumptions

List assumptions explicitly.

### Gaps

List unresolved information required before implementation.

## Verification

Before returning `ResourceDesign`, verify:

1. Every infrastructure requirement has been considered.
2. Existing resources were evaluated for reuse.
3. Existing stacks and constructs were evaluated for extension.
4. Every resource has a classification.
5. Resource relationships are explicit.
6. Resource dependencies are identified.
7. Existing IAM roles were evaluated for reuse.
8. New IAM roles have justification.
9. Target stack and construct are identified where possible.
10. Assumptions and gaps are explicitly documented.
11. No CDK code is generated.

## Scope Discipline

`additionalInformation` from the Input Manifest is the authoritative scope filter for the
run, and it reaches this stage through `RequirementsModel.scope`.

- Cover only what `scope.inScope` contains. Record anything else as deferred or
  out-of-scope; never silently include it because it shares a stack or feature.
- If `scope.authoritativeScopeFilter` is empty, blank, or a generic non-restrictive
  placeholder (for example "go as per the requirements"), there is NO restriction: cover
  everything the RequirementsModel identifies as in scope rather than narrowing further.
- Copy literal names, identifiers and ARN patterns from the source documents verbatim.
  Never invent one; anything not given becomes a declared gap.
- Report gaps as structured objects with `id`, `field`, `description`,
  `requiresHumanInput` and `blocksCodeGeneration` — never as free-text strings.

## Output Location

Write the result as valid JSON (no markdown or prose wrapper) to `workflow_output/CDKResourceDesignAgent.json`.

`workflow_output` lives at the workflow RUN ROOT: the directory that CONTAINS the cloned
repository's `src/` folder. It must never be created inside `src/`. The working directory
may already be `.../src`, so resolve it first rather than using a bare relative path:

```text
ROOT="$(pwd)"; case "$ROOT" in */src) ROOT="$(dirname "$ROOT")";; esac
mkdir -p "$ROOT/workflow_output"
```

Write to `$ROOT/workflow_output/CDKResourceDesignAgent.json` and, if reporting the location back, report the
full absolute path. Never emit an unsubstituted placeholder such as `<ROOT>`. When reading
a file from this folder, try `workflow_output/<file>` and fall back to
`../workflow_output/<file>`; ignore any stale copy under `src/workflow_output/`.

Writing the file is a side effect. Your final answer text must literally BE the complete
`ResourceDesign` JSON object — never a summary, a narrative, or an acknowledgement such as
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
