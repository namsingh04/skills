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