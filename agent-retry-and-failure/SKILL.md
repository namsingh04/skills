---
name: agent-retry-and-failure
description: How to retry, how much, and how to report a failure so the run can recover. Covers the two-retry cap and its durable ledger, telling a startup failure apart from a work failure, and the four fields every failure must carry. Attached to every agent node.
---

# Retry and failure

## The cap: two retries per agent

Three attempts total, then stop. This is a hard ceiling, not a target — most stages should
use zero.

The cap is per agent, not per stage. If a manager's validator fails one worker out of six,
that worker has two retries; the five that passed have spent nothing and are not re-run.

**Never re-run a sub-agent that succeeded.** Its output is already on disk and already
consistent with what other agents may have consumed. A second run that differs is worse than
no run.

### The ledger

`<run root>/workflow_output/_run/retry-ledger.json` holds what has actually been spent:

```json
{"schemaVersion":"1.0","maxRetriesPerAgent":2,
 "agents":{"CoreImplementationAgent":{"attempts":2,"reasons":["missing error path","import cycle"]}}}
```

Read it before you retry. Write it after. It exists because a manager that loses context
between turns would otherwise start the budget over, and because a resumed run must not
hand a failing agent a fresh three attempts on top of the three it already burned.

Increment `attempts` **before** the retry, not after. An attempt that crashes without
returning still counts, and a ledger updated only on completion never records the failures
that matter.

## Two kinds of failure, opposite responses

Tell these apart before spending a retry.

**WORK failure** — it ran, made tool calls, and returned something wrong, incomplete or
malformed.
→ Retry immediately, passing the *specific* defects. A retry that repeats the original
instruction unchanged will produce the original output unchanged.

**STARTUP failure** — it returned an error before making a single tool call: "Agent failed",
"Client config service request failed", a 401, an authorization or configuration error, or an
empty result within seconds of starting.
→ Nothing about the task caused this and nothing about the task will fix it. It is a
platform-side outage affecting every agent trying to start right now, and it lasts **minutes**.
Run `sleep 180`, then retry **once**. Waiting costs three minutes of wall clock and zero
tokens; retrying into the outage costs your only retry and is certain to fail. On a previous
run a manager retried five seconds after the first 401 and the retry died the same way,
forty-two seconds into an outage that was still going.

If the retry after the wait also fails: do not keep trying, and **do not start the other
sub-agents into the same window**. Carry on with the outputs you have, record in `warnings`
that the sub-agent could not be *started* and that this was a platform failure rather than a
defect in its task, and report `PARTIAL`.

Never report a startup failure as though the agent had done its work and found nothing.
That reads downstream as "there was nothing to find", which is a different and much more
expensive mistake.

## A retry must change something

Before you retry, be able to say what will be different. Give the agent:

- the specific validation errors, quoted,
- the exact file path it must write to,
- what it got wrong last time, in its own terms.

If nothing about the invocation changes, do not spend the retry — record the failure instead.

## When the budget is exhausted

Write your output with:

```json
{
  "status": "BLOCKED",
  "nextAction": "SKIP_DOWNSTREAM",
  "failure": {
    "stage": "codegen.core",
    "reason": "CoreImplementationAgent produced no output after 3 attempts; last error was a context-limit failure",
    "retryable": false,
    "recommendedAction": "re-run in resume mode; completed stages will be skipped and only this agent re-executed"
  },
  "attempt": 3,
  "retryBudget": {"used": 2, "max": 2}
}
```

Then **stop the dependent flow, not the whole run**. Downstream stages read `BLOCKED`, write
`SKIPPED`, and the run still reaches the summary — which is the only way anyone finds out
what happened.

## Every failure carries four things

Non-negotiable, in the `failure` object:

| Field | What it must contain |
|---|---|
| `stage` | The stage that failed, in `area.component` form. Not "the workflow". |
| `reason` | What actually happened, specifically. Quote the error. "Validation failed" is not a reason; "RequirementsModel.json had no `acceptanceCriteria` key" is. |
| `retryable` | Whether re-running the same thing could plausibly succeed. A missing input file is not retryable. A timeout is. |
| `recommendedAction` | What a human should do next, concretely enough to act on without reading the logs. |

Guessing here is worse than useless. If you do not know whether something is retryable, say
what you observed and mark it retryable — a wasted retry is cheaper than a run abandoned by
mistake.

## Things that are not failures

- An input that was never supplied. Record it, raise a gap, continue.
- A `PARTIAL` upstream. Work with the holes.
- A gap you could not resolve. That is what the review stage is for.
- A validation check that was skipped because the toolchain has no such command.

Reserve failure reporting for the run actually being unable to proceed.
