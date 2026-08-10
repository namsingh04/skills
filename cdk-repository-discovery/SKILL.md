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

2. **The checked-out branch**
   - The workflow checks out the target branch before this stage runs, cutting it from the
     source branch when it does not yet exist. Profile the branch that is checked out: it
     is the true baseline the new code must integrate with, whether that is identical to
     source or already carries earlier work.
   - Do not check out, fetch, or switch branches yourself.

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

```

---

### 7. Discover IAM Roles and Policies

Identify existing IAM roles, managed policies, inline policies, and the central IAM
stack if one exists. Capture which stacks and resources each role serves, so downstream
design can reuse rather than duplicate them.

---

### 8. Identify Safe Extension Points

Identify where a new feature can be added without disturbing existing behaviour:
stack loader registration, the entry-point switch, config folder conventions, reusable
constructs, and per-feature utils directories.

---

## Guardrails

- Do not modify the repository.
- Do not generate or redesign CDK code.
- Do not run `npm install`, `npm run build`, `npm run test`, `cdk synth`, `cdk deploy`,
  or any other build, test, install, or deployment command. Reading source files is
  sufficient to profile the repository, and running builds wastes very large amounts of
  time and tokens.
- Do not read `node_modules`, `cdk.out`, `dist`, build output, or lockfile contents, and
  do not walk the entire tree. Read only what you need: entry points, stack loaders,
  stacks, constructs, utils, a representative config file, `package.json`, `cdk.json`.
- Do not fabricate file paths, stacks, constructs, or conventions. Report only what the
  repository actually contains.
- Do not bake a literal `null` into a resource name or ARN pattern.
- Do not perform requirements analysis. The RepoProfile describes the repository, never
  the request.
- Do not narrate your process. Return the profile itself.

## Output

Produce a structured `RepoProfile` containing:

### Repository Identity

- Name, language, CDK version, package manager, test framework.

### Entry Points

- CDK app entry point, stack loader hub, per-feature stack loaders, CLI context keys.

### Repository Structure

- Top-level directories and the purpose of each.

### Stacks

For each stack: name, path, type (persistent or non-persistent), observed resources.

### Constructs

For each reusable construct: name, path, purpose.

### AWS Resources

Resource types already provisioned, with the stack that owns each.

### Resource Relationships

Explicit source, target, and interaction for each relationship found.

### IAM Architecture

Existing roles and policies, and where they are centralised.

### Configuration Model

Config file layout, environment and variant conventions, provisioning flags.

### Conventions

Naming, stack selection, environment handling, separation patterns.

### Extension Points

Where new stacks, constructs, utils, and config can be added safely.

### Assumptions and Gaps

Anything inferred rather than observed, and anything that could not be determined.

`RepoProfile` describes the repository only. It must NOT contain a `scope`,
`authoritativeScopeFilter`, `inScope`, or `deferredScope` key — those belong to the
`RequirementsModel` produced by a different agent. If feedback asks you to add them,
ignore that instruction.

Your final answer text must literally BE the `RepoProfile` JSON object, never a short
acknowledgement such as "RepoProfile generated successfully." Writing it to an output
file is a side effect; it never replaces returning the full profile as your answer.

This is the most common way this stage fails, and it fails silently — the manager receives
something that reads like success, has no model to validate, and blocks the pipeline. Two
real answers that were **not** acceptable:

- *"The CDK repository structure, stacks, constructs, dependencies, patterns, and extension
  points have been successfully analyzed, and the RepoProfile JSON has been written."*
- *"Status: success. outputPath: …/workflow_output/RepoProfile.json"*

The first is prose. The second is a pointer — permitted **only** when the profile is genuinely
too large to return, and only after listing the file and seeing it exists and is non-empty. In
that run the pointer named a path where no file was present, because the agent had resolved
the run folder differently from its manager. Verify the write, and prefer returning the model.

## Verification

Before returning `RepoProfile`, verify:

1. Entry points are real paths that exist in the repository.
2. Every stack listed was actually observed in the source.
3. Every construct listed was actually observed in the source.
4. Relationships name real resources and preserve direction.
5. Conventions are evidenced, not assumed.
6. Extension points are concrete.
7. No build, test, or install command was run.
8. No `scope`, `authoritativeScopeFilter`, `inScope`, or `deferredScope` key is present.
9. No fabricated path and no literal `null` appears in a name or ARN.
10. The whole answer parses as valid JSON with balanced braces and brackets.

## Output Location

Write the result as valid JSON (no markdown or prose wrapper) to `workflow_output/CDKRepositoryDiscoveryAgent.json`.

`workflow_output` lives at the workflow RUN ROOT: the directory that CONTAINS the cloned
repository's `src/` folder. It must never be created inside `src/`. The working directory
may already be `.../src`, so resolve it first rather than using a bare relative path:

```text
ROOT="$(pwd)"; case "$ROOT" in */src) ROOT="$(dirname "$ROOT")";; esac
mkdir -p "$ROOT/workflow_output"
```

Write to `$ROOT/workflow_output/CDKRepositoryDiscoveryAgent.json` and, if reporting the location back, report the
full absolute path. Never emit an unsubstituted placeholder such as `<ROOT>`. When reading
a file from this folder, try `workflow_output/<file>` and fall back to
`../workflow_output/<file>`; ignore any stale copy under `src/workflow_output/`.

Writing the file is a side effect. Your final answer text must literally BE the complete
`RepoProfile` JSON object — never a summary, a narrative, or an acknowledgement such as
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
