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

Write the result as valid JSON (no markdown or prose wrapper) to `workflow_output/CDKIntegrationAnalysisAgent.json`.

`workflow_output` lives at the workflow RUN ROOT: the directory that CONTAINS the cloned
repository's `src/` folder. It must never be created inside `src/`. The working directory
may already be `.../src`, so resolve it first rather than using a bare relative path:

```text
ROOT="$(pwd)"; case "$ROOT" in */src) ROOT="$(dirname "$ROOT")";; esac
mkdir -p "$ROOT/workflow_output"
```

Write to `$ROOT/workflow_output/CDKIntegrationAnalysisAgent.json` and, if reporting the location back, report the
full absolute path. Never emit an unsubstituted placeholder such as `<ROOT>`. When reading
a file from this folder, try `workflow_output/<file>` and fall back to
`../workflow_output/<file>`; ignore any stale copy under `src/workflow_output/`.

Writing the file is a side effect. Your final answer text must literally BE the complete
`IntegrationDesign` JSON object — never a summary, a narrative, or an acknowledgement such as
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
