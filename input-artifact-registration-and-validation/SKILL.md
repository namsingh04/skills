---
name: "input-artifact-registration-and-validation"
description: "Defines how to validate, classify, register, and trace workflow input artifacts without interpreting their business or technical content."
version: 1
created: "2026-07-27"
updated: "2026-07-27"
---

## When to Use

Use when initializing a CDK infrastructure generation workflow from Solution Design documents, Jira templates, acceptance criteria, standards, or other project artifacts.

## Procedure

1. Identify every provided input artifact.
2. Classify each artifact by purpose.
3. Register each artifact with a stable artifact ID.
4. Preserve the original artifact name and content reference.
5. Validate required artifact types are present.
6. Record missing or unreadable artifacts as gaps.
7. Do not interpret requirements, business logic, architecture, or implementation details.
8. Create traceable references for downstream agents.

## Required Artifact Types

- Solution Design
- Acceptance Criteria / Jira Template
- Standard.md

## Rules

- MUST preserve source references.
- MUST NOT modify source content.
- MUST NOT infer missing requirements.
- MUST NOT generate architecture decisions.
- MUST NOT generate CDK code.
- MUST report missing required inputs.
- MUST distinguish missing inputs from content gaps.
- MUST return structured output.

## Verification

1. Confirm Solution Design is registered.
2. Confirm Acceptance Criteria is registered.
3. Confirm Standard.md is registered.
4. Confirm every artifact has an ID.
5. Confirm missing artifacts are explicitly reported.
6. Confirm no architecture or implementation decisions were introduced.