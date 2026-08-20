---
name: workflow-run-contract
description: The file contract every agent in this workflow obeys - where to read, where to write, the output envelope, the status values, the upstream gate, and the resume rule. Attached to every agent node. Load this before doing any stage work.
---

# Workflow run contract

Nothing in this workflow passes in memory. You read named files and you write exactly one
file. Another agent will read what you write without ever seeing your reasoning, so the file
is the whole of your output.

## Where things are

The run root is the directory **above** your working directory. Your working directory is
usually the repository checkout, so the run root is normally `..`.

```
<run root>/workflow_output/
  _run/            run-config.json  retry-ledger.json  run-manifest.json
  00-inputs/       Input-Manifest.json  Solution-Model.json  Jira-Model.json
                   Standards-Profile.json  Additional-Instruction.json
  10-analysis/     Business.json  Functional.json  NonFunctional.json  Technical.json
                   Repo-Profile.json  Analysis-Validation.json
  20-spec/         Infrastructure.json  Integration.json  BusinessLogic.json
                   Implementation-Spec.json  Test-Fixtures.json  Spec-Validation.json
  30-gaps/         Gaps-For-Review.json  Gap-Resolutions.json  Gap-Decision.json
  40-codegen/      Generated-Files.json  Code-Validation.json  Code-Fix.json
  50-validation/   Toolchain-Profile.json  Validation-Result.json
  90-summary/      Traceability-Matrix.json  Run-Summary.json
  logs/            <stage>/…  raw command output, transcripts
```

The tree already exists. A script created every folder before any agent started, so you never
create a directory and never invent a path.

**Write to the absolute path you were given.** Do not derive it, do not rename it, do not add
a suffix. Every time an agent has derived its own output path, the file landed inside the
checkout under a name nobody was looking for, and the stage that needed it reported the input
as missing.

`workflow_output` lives **above** the checkout and never inside it. If you find yourself
writing to `src/workflow_output/…`, you have resolved a relative path against the wrong
directory.

Raw command output, long transcripts and tool dumps go in `logs/<stage>/`. Keep the envelope
small enough to read.

## The output envelope

Your file is a single JSON object with this shape. Fields you have nothing to say about are
present and empty, not absent.

```json
{
  "schemaVersion": "1.0",
  "stage": "analysis.functional",
  "agent": "FunctionalRequirementAgent",
  "runId": "", "branch": "", "mode": "create|resume|amend",
  "status": "OK|PARTIAL|BLOCKED|SKIPPED",
  "upstreamStatus": "",
  "nextAction": "CONTINUE|SKIP_DOWNSTREAM|RETRY|HUMAN_INPUT",
  "inputs": [{"path": "", "sha256": ""}],
  "outputPath": "",
  "payload": {},
  "gaps": [],
  "traceability": [],
  "failure": {"stage": "", "reason": "", "retryable": true, "recommendedAction": ""},
  "attempt": 1,
  "retryBudget": {"used": 0, "max": 2},
  "warnings": [], "errors": [],
  "metrics": {"startedAt": "", "endedAt": "", "model": ""}
}
```

Your stage-specific content goes in `payload` and nowhere else. Everything outside `payload`
means the same thing for every agent, which is what lets scripts read all of them.

`branch` is always what `git rev-parse --abbrev-ref HEAD` returns right now. Never a branch
name you read out of another file.

**Your final answer is this JSON object.** Not a prose summary of it, not a description of
where you put it.

## Status values

| Status | Meaning |
|---|---|
| `OK` | You did your work and produced everything you are responsible for. |
| `PARTIAL` | You produced usable output with known holes. The holes are in `gaps` or `warnings`. This is a normal, successful outcome. |
| `BLOCKED` | You could not produce usable output. Your input was missing, empty or unparseable, or your own tool calls failed past the retry budget. |
| `SKIPPED` | You did not run because upstream was BLOCKED or SKIPPED, or because your output already existed. |

`BLOCKED` is for **your own unrecoverable failure**. A gap count, a severity judgement, a
low-confidence answer or a downstream readiness flag never produces `BLOCKED`. Prefer
`PARTIAL` and say what is missing: a stage that blocks takes the whole run with it.

## The upstream gate — do this first

Read the `status` of the file you depend on:

- `OK` or `PARTIAL` → **proceed.** `PARTIAL` means work with the gaps you were given. It is
  never a reason to stop.
- `BLOCKED` or `SKIPPED` → write your own output with `"status":"SKIPPED"`, the
  `upstreamStatus` you saw, an empty payload and `"nextAction":"SKIP_DOWNSTREAM"`. Return it.
  Do not fail, do not raise.
- **You cannot find it, or cannot tell** → **fail open.** Proceed as if it were `OK` and
  record a warning saying so. Inability to see an upstream result is never grounds to block.
  A run has already been lost to a node that blocked while its upstream had in fact succeeded.

## The resume rule — the one definition of "already done"

This workflow can be resumed. A previous run's outputs may have been restored into
`workflow_output/` before you started, and your own file may already be sitting there.

> **A stage is COMPLETE if and only if its output file exists, parses as JSON, and its
> top-level `status` is `OK` or `PARTIAL`.**
> `BLOCKED`, `SKIPPED`, unparseable, truncated, empty or absent → **not complete. Do the work.**

Nothing else counts as evidence. Not that the file exists, not its size, not its timestamp.
A run killed mid-write leaves a file that exists and is wrong; `status` is the last thing an
agent writes, which is exactly why it is the test.

**Read your own output file before doing anything else.** If it is complete:

1. Return that JSON unchanged.
2. Add one `warnings` entry recording that you resumed from a restored output.
3. Rewrite every branch field from `git rev-parse --abbrev-ref HEAD`, and add a warning
   saying you corrected it. **A resumed run is on a different branch** — the run you are
   resuming died before it committed, so it never pushed its branch and there was nothing to
   check out. Any branch name in the restored file names a branch that does not exist here.
4. Stop.

Do not re-read your inputs, do not re-derive anything, and do not improve it. Another stage
may already have consumed that file, and a second version that differs is worse than no work
at all.

In `create` mode nothing was restored, the file will not be there, and this check costs one
read.

### The one exception: code generation also needs its files to still exist

A resume restores the previous run's **records**, never its **files**. The run being resumed
died before it committed, so its generated code existed only in a container that is gone.

Every stage before code generation produces records and nothing else, so the rule above is
complete for them. Code generation is different: `40-codegen/Generated-Files.json` can come
back with `status: OK` listing files that are not in this checkout.

> **Code generation is complete only if its status is `OK`/`PARTIAL` AND every file it lists
> still exists on disk.** One missing file makes the stage incomplete, and it regenerates.

Check the paths before trusting that record. Skipping regeneration because a status field says
so, when the files are gone, produces a run that reports success and commits nothing — the
staging step then fails with "nothing to commit" two stages later, and the cause is not
obvious from there.

**If you are a manager, decide this per sub-agent, not per stage.** A run that died halfway
leaves some outputs and not others, and the whole value of resuming is running only the ones
that are missing. Skipping an invocation costs nothing; invoking an agent so it can notice
its own output still starts an agent, still burns context, and still exposes the run to a
startup failure. Record in `warnings` which sub-agents you skipped and which you ran.

## Gaps

Any agent may raise a gap. The shape:

```json
{
  "id": "GAP-<stage>-<n>",
  "type": "AMBIGUITY|CONTRADICTION|MISSING|OUT_OF_SCOPE",
  "authoritySource": "solution|jira|standards|repository",
  "severity": "HIGH|MEDIUM|LOW",
  "requiresHumanInput": true,
  "blocksCodeGeneration": false,
  "question": "the question a reviewer can actually answer",
  "context": "what you read that produced it, with the source reference",
  "proposedDefault": "what happens if nobody answers",
  "resolution": ""
}
```

`resolution` is always empty when you write it. Only the human review fills it in.

## Traceability

Every requirement you act on, and every artifact you produce, gets an entry:

```json
{"requirementId": "FR-004", "source": "jira:ABC-123", "artifact": "20-spec/Integration.json#/interfaces/2", "evidence": "AC 3 states the 404 behaviour"}
```

This is assembled into a matrix that gets committed with the code. An entry you cannot
support with a source reference should not be written.

## Verify before you return

1. `write_file` succeeded, then **list the file** and confirm it is there and non-empty.
2. Re-read it and confirm it parses as JSON.
3. Confirm `status`, `stage`, `agent` and `outputPath` are all populated.
4. Confirm `outputPath` is the exact path you were given.

An agent that reports success without confirming the file exists is the single most common
way this workflow fails, because the failure surfaces two stages later as a missing input.
