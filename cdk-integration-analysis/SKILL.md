---
name: "cdk-integration-analysis"
description: "Defines how to analyze internal and external integrations for AWS CDK infrastructure, including API, event-driven, synchronous, asynchronous, authentication, contracts, and failure handling."
version: 1
---

## When to Use

Use when designing CDK infrastructure that integrates AWS resources, application components, or external services.

## Objective

Create an `IntegrationDesign` that clearly defines how systems and resources communicate and how integrations should behave.

## Inputs

- `RequirementsModel`
- `RepoProfile`

## Analysis

Analyze the requirements and repository profile to identify:

1. Internal resource-to-resource integrations.
2. External service integrations.
3. API interactions.
4. Event-driven interactions.
5. Synchronous communication.
6. Asynchronous communication.
7. Request and response flows.
8. Data and event flows.
9. Authentication and authorization boundaries.
10. Integration failure handling.
11. Retry and resilience requirements.
12. Existing integration patterns that should be reused.

## Integration Types

Identify and classify integrations as applicable:

- `API`
- `EVENT`
- `QUEUE`
- `SYNCHRONOUS`
- `ASYNCHRONOUS`
- `EXTERNAL_SERVICE`
- `DATABASE`
- `OTHER`

Do not force an integration into a category when the requirements do not support it.

## Internal Integrations

Identify communication between AWS resources and application components.

Examples:

- API Gateway → Lambda
- Lambda → EventBridge
- EventBridge → SQS
- SQS → Lambda
- Lambda → DynamoDB
- Lambda → Existing Service

For each integration, define:

- Source.
- Target.
- Integration type.
- Trigger.
- Data exchanged.
- Direction.
- Dependency.
- Required permissions.
- Failure behavior.

## External Integrations

Identify all external systems and services.

For each integration, define:

- External system.
- Calling component.
- Integration type.
- Purpose.
- Request information.
- Response information.
- Authentication requirement.
- Authorization requirement.
- Failure behavior.
- Retry behavior.
- Timeout requirements, if known.

Do not invent external systems or integration details.

## API Analysis

For API-based integrations, identify where possible:

- HTTP method.
- Endpoint or resource path.
- Request structure.
- Response structure.
- Headers.
- Authentication.
- Authorization.
- Validation.
- Error responses.

If information is unavailable, record it as a gap.

## Event Analysis

For event-driven integrations, identify:

- Event producer.
- Event source.
- Event bus or event mechanism.
- Event pattern.
- Event consumer.
- Event payload.
- Routing/filtering requirements.
- Delivery behavior.
- Failure handling.

## Queue Analysis

For queue-based integrations, identify:

- Producer.
- Queue.
- Consumer.
- Message structure.
- Visibility timeout requirements, if known.
- Dead-letter handling.
- Retry behavior.
- Ordering requirements, if applicable.

## Authentication and Authorization

Identify authentication and authorization boundaries for each integration.

Consider:

- Existing IAM roles.
- IAM policies.
- API authentication.
- External service credentials.
- Secrets.
- Tokens.
- Resource policies.

Prefer existing repository security patterns.

Do not invent authentication mechanisms.

## Failure and Resilience

Identify integration failure scenarios and required behavior:

- Retry.
- Timeout.
- Dead-letter queue.
- Error handling.
- Fallback.
- Circuit breaker.
- Idempotency.
- Duplicate event handling.

Only recommend mechanisms supported by requirements or repository standards.

## Existing Pattern Reuse

Identify existing integration patterns in `RepoProfile` that should be reused.

Prefer:

1. Existing integration constructs.
2. Existing event patterns.
3. Existing queue patterns.
4. Existing API patterns.
5. Existing authentication patterns.
6. Existing resilience patterns.

Do not introduce a new integration pattern when an appropriate existing pattern is available.

## Guardrails

- Do not generate CDK code.
- Do not modify the repository.
- Do not implement business logic.
- Do not assume Lambda is the integration point.
- Do not invent external services.
- Do not invent API contracts.
- Do not invent authentication mechanisms.
- Do not assume all integrations are synchronous.
- Explicitly distinguish synchronous and asynchronous flows.
- Preserve integration direction.
- Clearly identify assumptions.
- Clearly identify gaps.
- Do not make resource design decisions outside integration requirements.

## Output

Produce a structured `IntegrationDesign` containing:

### Internal Integrations

- Source.
- Target.
- Integration type.
- Trigger.
- Data exchanged.
- Direction.
- Dependencies.
- Permissions.
- Failure behavior.

### External Integrations

- External system.
- Calling component.
- Integration type.
- Purpose.
- Request.
- Response.
- Authentication.
- Authorization.
- Failure behavior.
- Retry behavior.

### API Flows

- Endpoint.
- Method.
- Request.
- Response.
- Authentication.
- Validation.
- Errors.

### Event Flows

- Producer.
- Event source.
- Event bus.
- Event pattern.
- Consumer.
- Payload.
- Routing.
- Failure handling.

### Queue Flows

- Producer.
- Queue.
- Consumer.
- Message.
- Retry.
- Dead-letter handling.
- Ordering.

### Resilience

- Retry.
- Timeout.
- Failure handling.
- Idempotency.
- Duplicate handling.

### Existing Patterns

List repository integration patterns that should be reused.

### Assumptions

List assumptions explicitly.

### Gaps

List unresolved integration information.

## Verification

Before returning `IntegrationDesign`, verify:

1. All internal integrations are identified.
2. All external integrations are identified.
3. Integration direction is explicit.
4. API, event, and queue flows are mapped where applicable.
5. Synchronous and asynchronous flows are distinguished.
6. Authentication and authorization boundaries are identified.
7. Failure and retry behavior is considered.
8. Existing integration patterns were evaluated for reuse.
9. Assumptions are clearly documented.
10. Gaps are clearly documented.
11. No CDK code is generated.