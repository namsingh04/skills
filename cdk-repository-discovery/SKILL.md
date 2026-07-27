---
name: "cdk-repository-discovery"
description: "Guidelines for analyzing an existing CDK repository to create a structured RepoProfile for extending infrastructure safely. Identifies repository structure, stacks, constructs, AWS resources, reusable patterns, existing IAM roles, dependencies, configuration, integrations, conventions, and extension points without modifying the repository or generating code."
version: 1
created: "2026-07-27"
updated: "2026-07-27"
---

## When to Use

Use this skill during the repository discovery stage of a reusable CDK infrastructure generation workflow.

Apply when:

- An existing CDK repository is provided through the Project Node.
- A source Git branch is available.
- New CDK infrastructure must be added to the existing repository.
- Existing repository structure and conventions must be preserved.
- Existing IAM roles should be reused where applicable.

The output of this skill is a structured `RepoProfile` consumed by the downstream Infrastructure Specification and CDK Code Generation stages.

This skill answers:

> "What already exists in the repository, and how should new infrastructure fit into it?"

It does not determine:

> "What new infrastructure is required?"

Requirements are handled by the `cdk-requirements-analysis` skill.

---

## Inputs

### Required

1. **Existing CDK Repository**
   - Accessed through the Project Node.
   - Represents the existing infrastructure codebase.

2. **Source Branch**
   - Defines the repository version to analyze.

### Optional Context

3. **Input Manifest**
   - Provides workflow context and Git information.

4. **Requirements Model**
   - If provided, may be used to focus discovery on relevant existing resources and patterns.
   - Do not change repository findings based on requirements.

---

## Primary Objective

Create a structured and accurate `RepoProfile` describing the existing CDK repository.

The RepoProfile must help downstream agents understand:

- Repository structure
- CDK application entry points
- Existing stacks
- Existing constructs
- Existing AWS resources
- Resource relationships
- Reusable constructs
- Existing IAM roles
- Dependencies
- Configuration patterns
- Environment handling
- Naming conventions
- Tagging conventions
- Repository coding patterns
- Existing integrations
- Extension points
- Potential files or constructs relevant to the new implementation

---

## Procedure

### 1. Identify Repository Structure

Inspect the repository structure and identify:

- CDK application entry point
- CDK app configuration
- Stack directories
- Construct directories
- Lambda directories
- Configuration directories
- Shared libraries
- Utility modules
- Test directories
- Deployment configuration
- Environment configuration
- Build configuration

Record relevant paths.

Do not assume standard CDK directory structures.

---

### 2. Identify CDK Entry Points

Identify:

- CDK application entry point
- Stack instantiation
- Environment configuration
- Context configuration
- Application configuration

Determine how stacks are instantiated and connected.

Capture the relevant file paths.

---

### 3. Discover Existing Stacks

Identify all CDK stacks.

For each stack capture:

- Stack name
- Source file
- Purpose
- Major resources
- Dependencies
- Constructs used
- Cross-stack references

Do not assume every CDK stack is independently deployable.

---

### 4. Discover Existing Constructs

Identify reusable constructs.

For each construct capture:

- Construct name
- Source file
- Purpose
- Resources created
- Input properties
- Output properties
- Dependencies
- Reusability

Identify constructs that may be extended or reused for new infrastructure.

Do not modify or redesign existing constructs.

---

### 5. Discover AWS Resources

Identify AWS resources already defined in the repository.

Examples include:

- API Gateway
- Lambda
- EventBridge
- SQS
- SNS
- DynamoDB
- S3
- Secrets Manager
- IAM
- Step Functions
- CloudWatch
- VPC resources

For each relevant resource capture:

- Resource type
- Logical identifier
- Construct or stack containing it
- Purpose, if identifiable
- Related resources
- Relevant source file

---

### 6. Discover Resource Relationships

Identify relationships between existing resources.

Capture:

- Lambda triggers
- EventBridge rules
- EventBridge targets
- SQS event sources
- API Gateway integrations
- DynamoDB access
- S3 notifications
- SNS subscriptions
- Cross-stack references
- Resource dependencies

Represent relationships explicitly.

Example:

```text
API Gateway
    └── integrates with ──► Lambda

Lambda
    └── publishes to ──────► EventBridge

EventBridge Rule
    └── targets ───────────► SQS

SQS
    └── triggers ──────────► Lambda

Lambda
    └── reads/writes ──────► DynamoDB