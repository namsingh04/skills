---
name: "cdk-analysis-orchestration"
description: "Routing, validation and retry policy for the CDK analysis stage. Coordinates the requirements and repository discovery sub-agents, gates their output through a validation sub-agent, retries only what failed, and hands a compact path-based result to the infrastructure specification stage or hard-stops the pipeline."
version: 1
created: "2026-07-30"
updated: "2026-07-30"
---

## When to Use

Use this skill in the orchestrator node that coordinates the CDK analysis stage.

It governs how sub-agents are sequenced, validated and retried. It does not perform
requirements analysis, repository discovery, or validation.

## Objective

Produce a validated analysis context for the infrastructure specification stage, or block
the pipeline when validation cannot be satisfied.

## Sub-Agent Execution

Round 1:

1. Run the requirements analysis and repository discovery sub-agents in parallel. Wait for
   both.
2. Run the validation sub-agent.

When invoking the validator you MUST paste the actual full model text into the task you
give it. It cannot see sub-agent results on its own. Without the real content it has
nothing to validate and will fail both models. This applies to every validation call, not
only the first.

## Retry Policy

While `validationStatus` is `"INVALID"`, and at most 3 retry rounds:

- Re-run only the agents named in `retryAgents`. Both in parallel if both are listed.
- Never re-run an agent whose model already passed. Carry its output forward unchanged.
- On a retry pass only two things: a pointer to the agent's original inputs, and that
  agent's own issues from the validator, condensed to at most 10 one-line bullets with no
  quoted evidence.
- Send each agent only its own issues. Never cross-feed one agent the other's issues.
- Never re-paste a previous invalid response. It adds size without helping and can push the
  retry over its context limit.
- Re-run the validator after each retry round, pasting in the new output plus any
  carried-forward passing output.

Any single agent therefore runs at most 4 times: once in round 1 plus up to 3 retries.
Track the rounds used and stop at 3.

## Sub-Agent Failure

If a sub-agent call itself errors — context overflow, crash, or no response — treat that
model as invalid and retry that agent, but never paste the raw error text to the validator
as if it were the model. State that the model is missing due to an upstream agent error.

A failed call counts as one used retry for that agent.

## Result Contract

When `validationStatus` is `"VALID"`:

```json
{
  "status": "OK",
  "requirementsModelPath": "<absolute path>",
  "repoProfilePath": "<absolute path>",
  "analysisValidation": {},
  "retryRoundsUsed": 0
}
```

Both paths must be fully-resolved absolute paths that exist on disk. Verify with `pwd` and
`ls` before returning. Never emit an unsubstituted placeholder such as `<ROOT>`.

Do NOT inline the model content. Each sub-agent already wrote its validated model to its
file, and those files are the hand-off. Inlining both models exceeds the output token limit
and returns truncated, unparseable JSON. If a file is stale or missing, write the validated
model there yourself.

When still `"INVALID"` after the 3rd retry round, hard stop:

```json
{
  "status": "BLOCKED",
  "stage": "CDK Analysis Orchestrator",
  "reason": "<what actually happened across the rounds>",
  "analysisValidation": {},
  "retryRoundsUsed": 3
}
```

Omit both paths when blocked, so nothing downstream mistakes a blocked result for usable
analysis. The `reason` must describe what really occurred — which model failed, what was
retried, what remained wrong — never a fixed canned phrase.

## Guardrails

- Do not perform requirements analysis, repository discovery, or validation yourself.
- Do not invoke the validator without pasting the actual current model content.
- Do not re-run an agent whose model already passed.
- Do not exceed 3 retry rounds.
- Do not return `status: "OK"` while the validator reports either model invalid.
- Do not hide validation issues. Always include the final verdict, OK or BLOCKED.
- Do not return a prose completion summary. The final answer IS the result JSON.

## Output Location

Write the result as valid JSON to `workflow_output/CDK-Analysis-Orchestrator.json`.

`workflow_output` lives at the workflow RUN ROOT: the directory that CONTAINS the cloned
repository's `src/` folder, never inside `src/`. Resolve it first:

```text
ROOT="$(pwd)"; case "$ROOT" in */src) ROOT="$(dirname "$ROOT")";; esac
mkdir -p "$ROOT/workflow_output"
```

Writing the file is a side effect; it never replaces returning the result JSON as the
answer.

## Verification

Before returning, verify:

1. The validator was called with real pasted model content on every round.
2. Only agents named in `retryAgents` were retried.
3. No more than 3 retry rounds were used.
4. `status` is `"OK"` only when the final verdict is `"VALID"`.
5. Both paths are absolute, resolved, and exist on disk.
6. No model content is inlined in the response.
7. The answer parses as valid JSON.
