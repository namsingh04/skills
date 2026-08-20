---
name: "run-summary-reporting"
description: "Report what a run actually did - stages complete, skipped and retried, gaps resolved and deferred, validation results, and where the code landed - without rounding anything up. Use when assembling or interpreting the execution summary."
version: 1
created: "2026-08-20"
updated: "2026-08-20"
---

# Run summary reporting

The summary is what anyone asks for when something went wrong, and it is the only artifact
that describes the run as a whole. It must be exactly true.

Most of it is assembled by a script, by counting. Where an agent contributes, the same
discipline applies: **the summary reports, it does not advocate.**

## Counting rules

**Stage completion uses the one rule.** A stage is complete if its output file exists, parses,
and `status` is `OK` or `PARTIAL`. Anything else is incomplete. Do not soften an incomplete
stage into a complete one because it "mostly worked", and do not count a `SKIPPED` stage as
complete because skipping it was correct.

**A resumed stage is complete, and it is also resumed.** Say both. A summary that shows twelve
complete stages without noting that seven were inherited from a previous run is misleading
about what this run actually did.

**Retries are counted from the ledger**, not from recollection. `_run/retry-ledger.json` is
the record.

**Gaps have three outcomes, not two:** resolved (the reviewer answered), deferred (the
reviewer saw it and chose not to answer, or left it blank), and unanswered (never reached a
reviewer). Collapsing deferred into resolved hides a decision nobody made.

## Report the things people go looking for

- **Where is the code?** Branch, commit sha, whether the push succeeded, the remote with any
  credentials stripped.
- **Did it push at all?** A run with `pushChanges` off completed successfully and shipped
  nothing. Say so plainly rather than reporting success and letting someone discover the
  absence.
- **What failed, and what should be done about it?** Every failure carries its stage, reason,
  whether it is retryable, and the recommended recovery action. That last field is what turns
  a summary into something actionable.
- **What was assumed?** Defaults the spec proceeded on, deviations the code agents recorded,
  gaps deferred. This is the list a reviewer should read before approving.
- **What did it cost?** Per-stage model and duration. Without it, nobody can tell which agent
  is burning the budget.

## Outcome

- `SUCCESS` — every stage complete, validation passed, changes pushed if a push was requested.
- `PARTIAL` — it produced usable output with known holes: a failing check, an open blocking
  gap, a stage that blocked, or a push that did not happen when one was requested.
- `FAILED` — no stage completed, or nothing was produced.

Be honest at the boundary. A run whose tests fail is `PARTIAL`, not `SUCCESS` with a note.

## Never fabricate

Everything in the summary comes from a file on disk. Branch names come from
`git rev-parse --abbrev-ref HEAD`, never from a restored output. File counts come from the
generation report. Validation results come from the validation result file.

If something is unknown, the value is `unknown` and that is a complete answer. An invented
commit sha or an assumed push status is worse than a blank, because it will be believed.

## Write for the person reading it at the end of the day

Alongside the JSON, produce something readable: outcome, branch, what to look at first, and
the open questions. Ordered by what someone needs to act on, not by pipeline order.

The person reading this has not been watching the run. Tell them what happened and what it
means, in that order.
