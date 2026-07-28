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