---
name: "workflow-run-contract"
description: "The file contract every agent in this workflow obeys - where to read, where to write, the output envelope, the status values, the upstream gate, and the resume rule. Attached to every agent node. Load this before doing any stage work."
version: 6
created: "2026-08-20"
updated: "2026-09-01"
---

# Workflow run contract

Nothing in this workflow passes in memory. You read named files and you write exactly one
file. Another agent will read what you write without ever seeing your reasoning, so the file
is the whole of your output.

## Your answer is raw JSON. No fence, no prose.

Read this before anything else, because it is the one mistake that makes correct work
unusable.

**Return the envelope object itself — starting with `{` and ending with `}`.** Not wrapped in
a ```` ```json ```` fence. Not preceded by "Here is the result". Not followed by a summary of
what you did. The first character of your answer is `{`.

Whoever called you parses your answer as JSON and reads `status` out of it. A manager deciding
whether to retry you, and the summary script deciding whether your stage completed, both do
this. A fence makes the parse fail, and a failed parse is indistinguishable from a stage that
produced nothing:

- On 2026-08-20 an ingestion stage did 11 minutes of correct work, wrote all five of its
  files, and returned `` ```json `` followed by a perfect envelope. Every automated reader of
  that answer saw unparseable text.
- Earlier the same day, four design agents returned prose reports — *"Here is a full structured
  summary of what was produced"* — instead of envelopes. Their files were fine. Their answers
  told the manager nothing it could act on.

If you want to explain yourself, put it in `warnings`. That field is read; prose around the
JSON is not.

**The one exception, and it is narrow.** If your own instruction tells you to answer with a
bare value -- a single word, a number, `true` or `false` -- because a conditional node reads
your answer directly, then that bare value IS your answer. Do not wrap it in an envelope, do
not add JSON around it, do not write a file. On 2026-08-20 an agent asked for the bare word
`true` followed this page instead of its own instruction, returned a 1126-character envelope,
and the conditional it fed could not evaluate it: *"There's a problem with your condition. You
wrote: {"schemaVersion":..."*. Everything upstream had succeeded. When your instruction and
this page disagree about the SHAPE of your answer, your instruction wins.

**And when you are the reader, be forgiving.** If an answer you received starts with a fence,
strip it and parse what is inside before concluding the agent failed. A correct envelope in a
fence is a formatting slip; treating it as a dead stage throws away real work and spends a
retry on an agent that had already succeeded. Be strict about what you emit and tolerant of
what you accept.


## Where things are

The run root is the directory **above** your working directory. Your working directory is
usually the repository checkout, so the run root is normally `..`. **But do not rely on that
guess.** The run root's ABSOLUTE path is written in `_run/run-config.json` as `runRoot`, and its
`workflow_output` as `outputRoot`; every output path you are handed is already absolute. Use the
absolute path — never a relative one.

```
<run root>/workflow_output/
  _run/            run-config.json  retry-ledger.json  run-manifest.json
  00-inputs/       Input-Manifest.json  Solution-Model.json  Jira-Model.json
                   Additional-Instruction.json
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

`workflow_output` lives **above** the checkout and never inside it — it is always a SIBLING of
the `src` checkout, at the run root. **NEVER write a relative `workflow_output`, and NEVER a path
inside the `src` checkout.** If you ever find yourself writing to `src/workflow_output/…`, you have
resolved a relative path against the wrong directory: use the absolute `outputRoot` from
`_run/run-config.json` (or the absolute path you were handed) instead. The only thing that belongs
inside `src` is the code this run must commit.

**Your runtime may tell you to keep newly created files inside `src/`.** That instruction is about
the repository CODE you generate for the commit. It does NOT apply to your stage-output envelope: that
ALWAYS goes to `<outputRoot>/<stage>/<file>` — a sibling of `src`, never inside it. When the two
seem to conflict, this contract wins for stage outputs.

**If the path you were handed still contains a placeholder — `<path>`, `<proven dir>`, `{…}`, or any
`<…>` token — it was not filled in for you.** Do NOT write to the literal token, and do NOT fall back
to `src`. Build the path yourself: take `outputRoot` from `_run/run-config.json`, add your stage
folder, add your exact filename. An unresolved token is the single most common reason an output lands
where the next stage cannot find it.

**Keep the stage folder a real directory.** Write `00-inputs/Solution-Model.json`, never a flattened
`00-inputs-Solution-Model.json`; the `/` is a directory separator, not part of the name.

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
  "runId": "", "branch": "<git -C <checkout> rev-parse --abbrev-ref HEAD>", "mode": "<the `mode` in _run/run-config.json>",
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

**`mode` is the value of `mode` in `_run/run-config.json` — read it from there.** It is one of
`create`, `resume`, `amend`. NEVER emit the literal string `create|resume|amend` (that is a schema
legend, not a value), and never guess `amend`. On 2026-08-25 thirty outputs on a `create` run
carried the placeholder or a guessed `amend` because they never read the config.

`branch` is the LIVE branch git reports for the CHECKOUT right now — a resumed run is on a freshly
cut branch, so a name read from a restored file would be wrong.

**Run git inside the CHECKOUT, with an ABSOLUTE path.** `workflow_output` is not a git repository,
so bare `git rev-parse` from there returns nothing and you record `no-git` or an empty string
(18 + 16 outputs did exactly that on 2026-08-25). The checkout is `<run root>/src`, where the run
root's absolute path is `runRoot` in `_run/run-config.json` (or the absolute path you were handed):

```
git -C <run root>/src rev-parse --abbrev-ref HEAD
```

Name the checkout explicitly with `-C`; never trust your working directory, never run bare `git`.
**Never record `no-git` or an empty branch** — if `git -C` still returns nothing, fall back to the
`branch` in `_run/run-config.json` and record a warning; otherwise record the failure in `warnings`
--
never a placeholder like `unknown` that reads downstream as if it were a branch name.

**Your final answer is this JSON object** — raw, unfenced, starting with `{`, as the top of
this page says. Not a prose summary of it, not a description of where you put it.

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
3. Rewrite every branch field from `git -C <checkout> rev-parse --abbrev-ref HEAD`, and a warning
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

## An input you cannot read is reported, never worked around

If a file you were told to read is not there, **say so in `warnings` and quote the exact path
you were given.** Then continue with what you do have, and mark the output `PARTIAL`.

Paths handed to you can be wrong. On 2026-08-20 four of eleven delegation messages contained a
mangled path — a duplicated directory segment spliced into the middle — and in two of them the
mangled one was `Solution-Model.json`, the most important input in the run. An agent that
quietly proceeds on the inputs it happened to find still produces a confident, complete-looking
answer, and nothing downstream can tell that its main source was missing.

Do not silently repair the path either. If you find the file somewhere else, say that too: a
mistyped path is a defect in the caller, and it stays invisible until someone reports it.

## Gaps

Any agent may raise a gap. The shape:

```json
{
  "id": "GAP-<stage>-<n>",
  "type": "AMBIGUITY|CONTRADICTION|MISSING|OUT_OF_SCOPE",
  "authoritySource": "solution|jira|repository",
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

## How to write the file

`write_file` takes exactly `{ "path": <string>, "content": <string> }`. **`content` is your file's
FULL TEXT as a single STRING** — if the file is JSON, serialise your envelope to a string first (the
characters `{`, `"`, `}` and so on, as text). **Never pass a JSON object, an array, or a nested
structure as `content`** — the tool rejects it with `Error: Invalid input format. Expected an object
with properties 'path' (string) and 'content' (string)`, and retrying the same way loops forever. A
large envelope once looped a whole stage this way (2026-08-31: BusinessLogic.json, 5 failed writes,
15 min, no output).

**A large output (roughly a screenful of JSON or more) must NOT go through `write_file` at all — write
it with `command_line` from the start.** A big `content` argument is where `write_file` fails: the tool
call itself gets truncated or malformed above a certain size, so the write is rejected with the error
above and the agent stalls retrying it — on 2026-09-01 `BusinessLogicDesignAgent` burned ~34 minutes
this way (two rejected single-shot writes, no output) and the delay pushed code generation past the
platform's auth window, failing the whole run. `command_line` is immune to the tool's shape and size
limits. Use a single-quoted heredoc to the ABSOLUTE path you were given:

```bash
cat > '/abs/path/you/were/given/File.json' <<'ENVELOPE_EOF'
{ ...your full envelope as text... }
ENVELOPE_EOF
```

**If the file is very large, write it in SEQUENTIAL APPENDED CHUNKS** so no single tool call carries the
whole payload — start with `cat > file <<'EOF' … EOF` for the first part, then `cat >> file <<'EOF' …
EOF` for each following part. Keep each chunk to roughly a screenful. This is what makes a big model
file (a full BusinessLogic / Infrastructure / Integration spec) land reliably instead of intermittently.

**And if `write_file` ever returns the format error (or any failure), do NOT retry it unchanged** —
switch to the `command_line` heredoc above immediately; never loop on `write_file`.

Then list the file to confirm it exists and is non-empty. The point is that the file lands, once, with
your real content — via `command_line` for anything large, `write_file` only for small files.

## Verify before you return

1. `write_file` succeeded (or the `command_line` fallback did), then **list the file** and confirm it is there and non-empty.
2. Re-read it and confirm it parses as JSON.
3. **Confirm it is the ENVELOPE, not a bare payload.** `stage`, `agent`, `status`,
   `outputPath` and `payload` must be at the TOP level, with your content inside `payload`.
   Writing your profile or model straight to the file — correct content, no envelope around
   it — is the failure a downstream gate cannot distinguish from a broken stage. It happened
   on 2026-08-20: a good repository profile was rejected because it had no envelope, and the
   whole analysis stage was failed and retried.
4. Confirm `outputPath` is the exact path you were given.

An agent that reports success without confirming the file exists is the single most common
way this workflow fails, because the failure surfaces two stages later as a missing input.
