---
name: "cdk-infrastructure-specification-manager"
description: "Coordinates infrastructure specification for extending existing AWS CDK repositories by parallelizing resource design, integration analysis, and business logic analysis, then synthesizing the results into a complete InfraSpec."
version: 1
---

## When to Use

Use when adding new AWS CDK infrastructure to an existing CDK repository.

## Objective

Create a complete `InfraSpec` that defines how the required infrastructure fits into the existing repository.

## Inputs

- `RequirementsModel`
- `RepoProfile`
- `standard.md`
- `repository`
- `sourceBranch`
- `targetBranch`

## Sub-Agent Execution

Run these sub-agents in parallel:

1. `CDKResourceDesignAgent`
2. `CDKIntegrationAnalysisAgent`
3. `CDKBusinessLogicAnalysisAgent`

Provide each sub-agent only the relevant requirements and repository context needed for its analysis.

Wait for all three sub-agents to complete before synthesis.

## CDKResourceDesignAgent

Analyze:

- Required new AWS resources
- Existing resources that can be reused
- Existing stacks and constructs to extend
- Resource configuration
- Resource dependencies
- Resource relationships
- Existing IAM roles to reuse

## CDKIntegrationAnalysisAgent

Analyze:

- External service integrations
- API interactions
- Event-driven integrations
- EventBridge routing
- SQS interactions
- Input/output contracts
- Authentication requirements
- Integration failure handling
- Retry and resilience requirements

## CDKBusinessLogicAnalysisAgent

Analyze:

- Business logic location
- Conditions and branching
- Validation
- Data transformation
- Error handling
- Retry requirements
- Business rules across resources

Do not assume business logic belongs in Lambda. Determine the appropriate location based on requirements and architecture.

## Synthesis

After all sub-agents complete, synthesize:

- `RequirementsModel`
- `RepoProfile`
- `standard.md`
- Git context
- Resource Design findings
- Integration Analysis findings
- Business Logic findings

Resolve overlaps and inconsistencies using the requirements, repository structure, and standards as the primary sources.

## Infrastructure Decisions

The `InfraSpec` MUST explicitly identify:

### Repository Integration

- Target stack
- Target construct
- Files to create
- Files to modify
- Existing patterns to follow

### Resource Strategy

For every resource, classify it as:

- `REUSE`
- `EXTEND`
- `CREATE`
- `REFERENCE`

### Resource Relationships

Explicitly define:

- Resource-to-resource connections
- Dependencies
- Invocation relationships
- Event flows
- Data flows

### IAM Strategy

Identify:

- Existing IAM roles to reuse
- Required permissions
- New IAM resources only when necessary

### Business Logic

Identify:

- Logic location
- Conditions
- Validation
- Transformations
- Error handling
- Retry behavior

### Integrations

Identify:

- External services
- API calls
- Event integrations
- Authentication
- Request/response contracts
- Failure handling

### Configuration

Identify:

- Environment variables
- CDK context
- Configuration files
- Environment-specific values

### Acceptance Criteria

Map each acceptance criterion to one or more infrastructure or implementation decisions.

### Gaps

Identify unresolved requirements or missing information.

Never invent requirements. Clearly mark assumptions and gaps.

## Guardrails

- Do not generate CDK code.
- Do not modify the repository.
- Do not create duplicate resources when existing resources can be reused.
- Prefer existing stacks and constructs where appropriate.
- Reuse existing IAM roles when applicable.
- Do not invent missing requirements.
- Do not assume business logic belongs in Lambda.
- Follow existing repository structure and CDK conventions.
- Ensure all resource relationships are explicit.
- Ensure all acceptance criteria are addressed.
- Preserve unresolved gaps in the final output.

## Output

Produce a complete `InfraSpec` containing:

- Repository integration plan
- Resource strategy
- Target stack and construct
- Files to create or modify
- Existing resources to reuse
- New resources
- Resource relationships
- Dependencies
- IAM strategy
- Business logic placement
- External integrations
- Configuration
- Environment requirements
- Acceptance criteria mapping
- Implementation constraints
- Assumptions
- Remaining gaps

## Verification

Before returning `InfraSpec`, verify:

1. All requirements are mapped.
2. All acceptance criteria are addressed.
3. Existing resources were considered for reuse.
4. Existing IAM roles were considered for reuse.
5. Resource relationships are explicit.
6. Business logic placement is defined.
7. External integrations are identified.
8. Repository conventions are followed.
9. Gaps and assumptions are clearly identified.
10. No CDK code is generated.

## Upstream Block Gate

The analysis context arrives from the CDK Analysis Orchestrator and carries a `status`.

Check it before doing anything else.

- If `status` is `"BLOCKED"`, stop. Do not invoke any sub-agent and do not attempt to build
  an `InfraSpec` from unvalidated analysis. Return exactly:

```json
{
  "status": "BLOCKED",
  "stage": "CDK Infrastructure Specification Agent",
  "upstreamReason": "<the orchestrator's reason>",
  "analysisValidation": {}
}
```

  and write that same object to the output file.

- Only when `status` is `"OK"` proceed.

The context is compact: it carries `requirementsModelPath` and `repoProfilePath`, not the
models themselves. Read both models yourself with read_file from those paths before
starting work.

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

Write the result as valid JSON (no markdown or prose wrapper) to `workflow_output/CDK-Infrastructure-Specification-Agent.json`.

`workflow_output` lives at the workflow RUN ROOT: the directory that CONTAINS the cloned
repository's `src/` folder. It must never be created inside `src/`. The working directory
may already be `.../src`, so resolve it first rather than using a bare relative path:

```text
ROOT="$(pwd)"; case "$ROOT" in */src) ROOT="$(dirname "$ROOT")";; esac
mkdir -p "$ROOT/workflow_output"
```

Write to `$ROOT/workflow_output/CDK-Infrastructure-Specification-Agent.json` and, if reporting the location back, report the
full absolute path. Never emit an unsubstituted placeholder such as `<ROOT>`. When reading
a file from this folder, try `workflow_output/<file>` and fall back to
`../workflow_output/<file>`; ignore any stale copy under `src/workflow_output/`.

Writing the file is a side effect. Your final answer text must literally BE the complete
`InfraSpec` JSON object — never a summary, a narrative, or an acknowledgement such as
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
