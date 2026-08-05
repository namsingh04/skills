---
name: "cdk-input-artifact-registration"
description: "Guidelines for validating, classifying, and registering workflow input artifacts and Git context for reusable CDK infrastructure generation workflows. Produces a compact, traceable Input Manifest without performing requirements analysis, repository analysis, infrastructure design, or code generation."
version: 1
created: "2026-07-27"
updated: "2026-07-27"
---

## When to Use

Use this skill at the beginning of a CDK infrastructure generation workflow after the
UI/Input Node has collected the required inputs.

Extraction itself is delegated. Use `confluence-document-extraction` for a Confluence
source document, `jira-issue-extraction` for issue-sourced acceptance criteria,
`structured-table-extraction` for any table whose values downstream stages read field by
field, and `mermaid-diagram-interpretation` for any diagram that carries architectural
meaning. This skill governs what is registered and how it is verified, not how each source
is read.

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
   - Sourced from one or more issue keys, supplied comma-separated and processed in the
     order given. There is no file fallback and no alternative source: never derive
     criteria from the Solution Design, read them off disk, or reuse a previous run's.
   - A blank key list is a recorded gap, not a failure: register an empty extraction, name
     the missing input in `warnings` and `gaps`, and continue.
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
```

`extractedText` must carry the artifact's real, fully decoded text — not a path, not a
placeholder, and not a description of the content. A reference alone is not registration.

`additionalInformation` must be preserved verbatim exactly as supplied, whatever it says.
Never interpret, summarise, normalise, or substitute your own judgement for it, and never
drop it when it is blank or `NA` — the requirements stage decides what it means, and it
must see precisely what the user typed. Registration never resolves that question.

When an artifact lives in Confluence rather than as an uploaded file, register it the same
way, with `source` set to `confluence` and the page id recorded, using the fetched page
body as `extractedText`.

## Guardrails

- Do not perform requirements analysis, repository discovery, or infrastructure design.
- Do not interpret what the extracted text means. Extraction is required; interpretation
  belongs to the requirements stage.
- Do not treat a raw byte read of a PDF as extraction. PDFs are binary: decode them with a
  real converter. If the result starts with `%PDF-`, contains `stream`, `endstream`, or
  `FlateDecode`, or is largely non-printable, the extraction failed — mark the artifact
  `unreadable` with a reason rather than passing binary through.
- Do not invent, summarise, or fabricate content for an artifact that could not be read.
- Do not mark an artifact missing merely because it arrived by a different route than
  expected.
- Report every missing or unreadable input explicitly.

## Verification

Before returning the `InputManifest`, verify:

1. Every required artifact is registered with a status.
2. Every available artifact carries real decoded `extractedText`.
3. No `extractedText` contains binary or container markers.
4. `additionalInformation` is present and byte-for-byte identical to the input.
5. Git source and target branches are recorded.
6. `validation.missingInputs` lists everything absent or unreadable.
7. The whole answer parses as valid JSON with balanced braces and brackets.

## Output Location

Write the result as valid JSON (no markdown or prose wrapper) to `workflow_output/Input-Artifact-Registration.json`.

`workflow_output` lives at the workflow RUN ROOT: the directory that CONTAINS the cloned
repository's `src/` folder. It must never be created inside `src/`. The working directory
may already be `.../src`, so resolve it first rather than using a bare relative path:

```text
ROOT="$(pwd)"; case "$ROOT" in */src) ROOT="$(dirname "$ROOT")";; esac
mkdir -p "$ROOT/workflow_output"
```

Write to `$ROOT/workflow_output/Input-Artifact-Registration.json` and, if reporting the location back, report the
full absolute path. Never emit an unsubstituted placeholder such as `<ROOT>`. When reading
a file from this folder, try `workflow_output/<file>` and fall back to
`../workflow_output/<file>`; ignore any stale copy under `src/workflow_output/`.

Writing the file is a side effect. Your final answer text must literally BE the complete
`InputManifest` JSON object — never a summary, a narrative, or an acknowledgement such as
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
