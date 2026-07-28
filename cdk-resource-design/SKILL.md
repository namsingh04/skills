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