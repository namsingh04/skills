---
name: "cdk-analysis-output-validation"
description: "Contract-based validation of the RequirementsModel and RepoProfile produced by the CDK analysis sub-agents. Performs fast, mechanical structural checks and returns a machine-readable verdict naming exactly which agent must be retried, without performing requirements analysis or repository discovery."
version: 1
created: "2026-07-30"
updated: "2026-07-30"
---

## When to Use

Use this skill as the quality gate inside the CDK analysis stage, after
CDKRequirementsAnalysisAgent and CDKRepositoryDiscoveryAgent have produced their models
and before the orchestrator hands anything to the infrastructure specification stage.

## Objective

Decide whether a usable `RequirementsModel` and `RepoProfile` were produced, and report a
structured verdict the orchestrator can act on.

This skill validates structure. It does not analyse, fix, or improve either model.

## Inputs

Both models are pasted into the task by the orchestrator. Validate exactly what is given.

If the orchestrator pastes newer content for a model, that pasted content is the current
one and takes precedence over anything referenced elsewhere.

## Speed Requirement

This is a pure JSON-shape check and must finish in seconds, not minutes.

- Do not call, load, or execute any skill — in particular not `cdk-requirements-analysis`
  or `cdk-repository-discovery`.
- Do not read repository or project files, clone anything, or inspect the codebase.
- Do not re-derive or cross-check the analysis against the source documents.

Everything needed is in the two JSON strings supplied. Inspect only those, then answer.

If a skill call somehow occurs and fails (for example "Invalid skill name provided"),
ignore it completely and continue. A skill failure is not a validation error: it must
never appear in the issues, and must never affect `validationStatus` or `retryAgents`.

## Scoping Rule

The scope contract belongs to the `RequirementsModel` ONLY.

Never require, expect, or check for `scope`, `authoritativeScopeFilter`, `inScope`, or
`deferredScope` on the `RepoProfile`. A RepoProfile has no scope object and must never be
failed for lacking one. Apply each checklist strictly to its own model.

## RequirementsModel Checks

Treat these as literal key-presence checks, not semantic judgement calls.

1. Parses as valid JSON — not prose, not bullet text, not markdown-wrapped, braces and
   brackets balanced, no unescaped code or ARN concatenation.
2. A top-level key literally named `scope` exists.
3. `scope.authoritativeScopeFilter` exists, is non-null and non-empty. Any other spelling
   or placement is a fail: `authoritativeFilter`, `scopeFilter`, `generatedFrom.scopeFilter`.
4. `scope.inScope` exists as a non-empty array.
5. `scope.deferredScope` exists under `scope`, not at the document root.
6. `scope.authoritativeScopeFilter` equals the InputManifest `additionalInformation` text
   verbatim.
7. Scope consistency:
   - If `additionalInformation` names specific resource types or features, `inScope` is
     limited to what it actually names, and `deferredScope` covers the rest of what the
     Solution Design mentions.
   - If `additionalInformation` is empty, blank, or a generic non-restrictive placeholder
     (for example "go as per the requirements"), there is NO restriction. Here a narrow
     `inScope` or a padded `deferredScope` is the defect: check instead that `inScope`
     covers everything described, and treat an empty `deferredScope` as correct.
8. Gaps are structured objects with `id`, `field`, `description`, `requiresHumanInput`,
   `blocksCodeGeneration` — not free-text strings.
9. Requirement and resource sections carry enough content for downstream InfraSpec
   generation.

Checks 2 to 5 must all hold simultaneously. A model satisfying some and not others fails
exactly like one satisfying none.

## RepoProfile Checks

1. Parses as valid JSON — not prose, not markdown-wrapped, balanced braces and brackets. A
   short acknowledgement such as "RepoProfile generated successfully." is an automatic
   fail, not a partial pass.
2. Content is concrete and evidenced: real entry points, stacks, constructs, conventions
   and extension points — not generic or empty placeholders.
3. No fabricated file paths or conventions.
4. No literal `null` baked into a resource name or ARN pattern.
5. No `scope`, `authoritativeScopeFilter`, `inScope` or `deferredScope` key is present.

## Guardrails

- Do not perform requirements analysis or repository discovery.
- Do not fix, rewrite, complete or improve either model — only report issues.
- Do not infer or reconstruct a missing output. A model you were not given is invalid, not
  acceptable.
- Do not mark a model valid because it merely looks complete; required keys must literally
  be present under their exact names.
- Do not re-invoke any sub-agent. Retry decisions belong to the orchestrator.
- Report every failing check in a single verdict. Never stop at the first error: if three
  keys are misplaced, list all three so one retry can fix them all.
- Keep every issue terse — the field plus one short sentence. Never include an `evidence`
  field, never quote the model back, never restate its content. The verdict is fed into a
  retry prompt, and long verdicts push that retry over its context limit.

## Handling a Missing or Errored Model

If a model arrives as an error string (for example
`Error: Cannot read properties of undefined`), empty, or absent, mark THAT model invalid
with the single reason "upstream agent produced no usable output" and list only THAT agent
in `retryAgents`.

Do not let it affect the judgement of the other model, which is still validated normally on
its own merits.

## Output

Your final answer text must literally BE this JSON object and nothing else — no prose
before or after:

```json
{
  "modelType": "AnalysisOutputValidationResult",
  "requirementsModel": {
    "valid": true,
    "issues": [{ "field": "<key path>", "reason": "<one short sentence>" }]
  },
  "repoProfile": {
    "valid": true,
    "issues": [{ "field": "<key path>", "reason": "<one short sentence>" }]
  },
  "validationStatus": "VALID",
  "retryAgents": [],
  "validationErrors": []
}
```

- `validationStatus` is exactly `"VALID"` or `"INVALID"` — never `"error"` or any other
  value, no matter what went wrong on your side.
- `validationStatus` is `"VALID"` only when BOTH models are valid.
- `retryAgents` lists only agents whose own model failed one of its own checks. It is an
  empty array when `validationStatus` is `"VALID"`. Never list both agents by default, and
  never list an agent because you had trouble running.

## Output Location

Write the result as valid JSON to `workflow_output/CDKAnalysisOutputValidationAgent.json`.

`workflow_output` lives at the workflow RUN ROOT: the directory that CONTAINS the cloned
repository's `src/` folder, never inside `src/`. Resolve it first:

```text
ROOT="$(pwd)"; case "$ROOT" in */src) ROOT="$(dirname "$ROOT")";; esac
mkdir -p "$ROOT/workflow_output"
```

Writing the file is a side effect; it never replaces returning the verdict JSON as your
answer.

## Verification

Before returning, verify:

1. `validationStatus` is exactly `"VALID"` or `"INVALID"`.
2. Both models were judged independently.
3. No RepoProfile issue references a scope key.
4. `retryAgents` contains only agents whose own model failed.
5. No skill failure influenced the verdict.
6. Every issue is one short sentence with no evidence field.
7. All failing checks are reported, not just the first.
8. The answer parses as valid JSON.
