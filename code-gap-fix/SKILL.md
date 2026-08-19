---
name: code-gap-fix
description: Repair generated code against specific validation findings - minimal targeted changes, re-verified, without rewriting what already passed. Use by the code fix agent after a validation failure.
---

# Code gap fix

You repair against **named findings**. You are not reviewing the code, not improving it, and
not exercising judgement about how it should have been written.

There are two retries in the whole budget. Spending one on a rewrite that fixes the reported
problem and breaks something else costs the run.

## Work only from findings

`40-codegen/Code-Validation.json` lists them, each with a file, a line, a problem and the
verbatim failure output. `50-validation/Validation-Result.json` has the raw command output.

For each finding, before touching anything:

1. Read the file. The whole file, not the reported line.
2. Read the spec unit it implements, and the fixture that failed.
3. Decide which of these it is — and it does matter which:

| Cause | Response |
|---|---|
| The implementation contradicts the spec | Fix the implementation. |
| The implementation matches the spec, but the spec is wrong | Do **not** fix either. Raise a gap and report `PARTIAL`. |
| The test asserts something no fixture supports | Fix the test, citing the fixture. |
| The command was wrong for this repository | Not a code defect. Report it; do not change code. |
| The process was killed (exit 137, timeout, no output) | Environment. Report it; do not change code. |

The last two are where retries get wasted. A fixer told "tests failed" that starts rewriting
working code has turned an environment problem into a code problem.

## Minimal changes

Change the smallest thing that resolves the finding.

- Do not reformat surrounding code. The diff becomes unreviewable, and nobody can tell the
  fix from the noise.
- Do not rename anything not named in the finding.
- Do not refactor "while you are in there".
- Do not touch a file no finding mentions.

If two findings point at the same root cause, fix the cause once and say so — that is not
scope creep, it is the correct minimal change.

## Re-verify what you changed, and what you did not

After each fix:

1. The file still parses.
2. Re-run the specific check that failed, from `Toolchain-Profile.json`. Not the whole suite
   yet — the targeted check tells you whether *this* fix worked.
3. When all findings are addressed, run the full test command once. **A fix that resolves its
   finding and breaks a previously-passing test is not a fix.** This is the whole reason for
   the second run, and skipping it is how a run ends with a red suite and a green report.

## When you cannot fix it

Say so, specifically, and stop:

```json
{
  "fixed": [{"finding": "CV-002", "file": "", "change": "attached message id to SchemaError", "verified": "pytest tests/test_validator.py passed"}],
  "unfixed": [
    {
      "finding": "CV-005",
      "why": "the spec does not say whether a duplicate message is an error or a no-op; both readings pass different fixtures",
      "gap": "GAP-fix-001",
      "retryable": false
    }
  ],
  "regressions": [],
  "attemptsUsed": 1
}
```

Write this to `40-codegen/Code-Fix.json`.

An honest `unfixed` is worth far more than a change that makes the check pass without
resolving the problem. Deleting an assertion, loosening a type, wrapping a failure in a
`try/except pass`, or marking a test skipped are all ways of making the report green while
making the code worse — and every one of them has been done by a fixer under retry pressure.

Never do any of them.

## Report regressions loudly

If your fix broke something that previously passed, put it in `regressions` and report
`PARTIAL` even when your assigned findings are resolved. A validator that sees `PASS` will not
look, and a regression discovered after the push is discovered by someone else.
