---
name: "cdk-input-artifact-registration"
description: "Guidelines for validating, classifying, and registering workflow input artifacts and Git context for reusable CDK infrastructure generation workflows. Produces a compact, traceable Input Manifest without performing requirements analysis, repository analysis, infrastructure design, or code generation."
version: 1
created: "2026-07-27"
updated: "2026-07-27"
---

## When to Use

Use this skill at the beginning of a CDK infrastructure generation workflow after the UI/Input Node has collected the required inputs.

Apply when the workflow receives:

- Solution Design document
- Jira template or Acceptance Criteria
- standard.md
- Source Git branch
- Target Git branch

The existing CDK repository is provided through the Project Node and should not be treated as a user-uploaded artifact.

The purpose of this skill is to create a reliable and traceable input contract for downstream workflow stages.

---

## Required Inputs

The workflow expects the following inputs:

1. **Solution Design**
   - Defines the proposed solution and architecture.
   - Register the original file reference.
   - Do not analyze or interpret the solution.

2. **Acceptance Criteria**
   - Defines expected behavior and functional requirements.
   - Register the original file reference.
   - Do not analyze or interpret the acceptance criteria.

3. **standard.md**
   - Defines standards to be followed during CDK implementation and code generation.
   - Register the original file reference.
   - Do not apply or interpret standards at this stage.

4. **Source Branch**
   - Git branch representing the existing implementation baseline.
   - Preserve the branch name exactly as provided.

5. **Target Branch**
   - Git branch where generated changes are intended to be applied.
   - Preserve the branch name exactly as provided.

---

## Procedure

### 1. Validate Input Availability

Verify that all required inputs are present.

Required:

- Solution Design
- Acceptance Criteria
- standard.md
- Source Branch
- Target Branch

If any required input is missing, mark the workflow as `blocked` and identify the missing input.

---

### 2. Classify Artifacts

Classify each file using the following artifact types:

| Input | Artifact Type |
|---|---|
| Solution Design | `solution_design` |
| Acceptance Criteria | `acceptance_criteria` |
| standard.md | `coding_standard` |

Do not invent additional artifact types unless required by the workflow.

---

### 3. Register Artifact References

For each artifact:

- Preserve the original file reference.
- Record the logical artifact name.
- Record the artifact type.
- Record availability status.
- Do not copy or reproduce the file contents into the manifest.
- Do not summarize the file contents.

The manifest should reference artifacts rather than embedding their contents.

---

### 4. Register Git Context

Record:

- Source branch
- Target branch

Preserve branch names exactly as provided.

Do not infer:

- Repository name
- Client name
- AWS account
- AWS region
- Stack name
- Deployment environment

These values must be obtained from later workflow stages or the Project Node when required.

---

### 5. Validate Basic Readiness

Determine whether the workflow has enough input information to proceed.

Use:

- `ready` when all required inputs are available.
- `blocked` when one or more required inputs are missing or unusable.

Do not perform deep content validation.

Detailed document analysis belongs to the Requirements Analysis stage.

Detailed repository validation belongs to the Repository Discovery stage.

---

### 6. Create Input Manifest

Create a compact and traceable `InputManifest`.

The manifest must contain:

- Registered artifacts
- Git source branch
- Git target branch
- Validation status
- Missing inputs, if any

The manifest must be suitable for consumption by downstream agents.

---

## Output Schema

Return the following logical structure:

```json
{
  "artifacts": [
    {
      "name": "solutionDesign",
      "type": "solution_design",
      "reference": "<file-reference>",
      "status": "available"
    },
    {
      "name": "acceptanceCriteria",
      "type": "acceptance_criteria",
      "reference": "<file-reference>",
      "status": "available"
    },
    {
      "name": "standard",
      "type": "coding_standard",
      "reference": "<file-reference>",
      "status": "available"
    }
  ],
  "git": {
    "sourceBranch": "<source-branch>",
    "targetBranch": "<target-branch>"
  },
  "validation": {
    "status": "ready",
    "missingInputs": []
  }
}