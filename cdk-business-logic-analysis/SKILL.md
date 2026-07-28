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