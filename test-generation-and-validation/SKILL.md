---
name: "test-generation-and-validation"
description: "Turn specified contracts and the fixtures extracted from the solution document into runnable tests in the repository's own framework, and interpret what a validation run actually proved. Use by the test authoring and code validation agents."
version: 3
created: "2026-08-20"
updated: "2026-08-24"
---

# Test generation and validation

## Tests come from fixtures, not from the implementation

`20-spec/Test-Fixtures.json` holds sample exchanges taken from the solution document. Those
are your assertions: real inputs with the outputs a human specified.

**Never write a test by reading the implementation and asserting what it does.** That test
passes by construction, proves nothing, and actively hides the defect it was supposed to
catch. If you find yourself running the code to find out what to assert, stop — the answer is
in the fixture, or it is a gap.

A test that cannot fail is worse than no test.

## Use the repository's framework and layout

From `10-analysis/Repo-Profile.json` and `50-validation/Toolchain-Profile.json`:

- the framework it already uses,
- where tests live and how files are named,
- how a typical test is structured — the profile quotes one; imitate it,
- how external calls are faked here: fixtures, factories, a mocking library, recorded
  responses.

Do not introduce a second test framework. Do not restructure the test suite.

Keep filesystem inspection scoped to the TARGET project's test directory and pipe `ls`/`find`
through `head`; do not scan the whole monorepo — a large listing is re-sent on every later turn and
is the biggest avoidable token cost here.

## What to cover

**Every fixture becomes at least one test**, named for the obligation it checks, citing the
requirement in a comment or the test name — whatever the repository's convention allows.

**Error paths matter as much as happy paths.** `Integration.json`'s `errorBehaviour` is
specified; a suite that only tests success has tested the half that was going to work anyway.

**A contract marked `sampleless: true`** gets a structural test — shape, required fields,
error type — not an invented behavioural assertion. Say in the test name that the expected
values were not specified.

Line coverage must exceed **85%** — the validation stage measures it and fails the run below
that. Reach it the right way: cover the error and edge paths, which is usually where the missing
coverage is, and where real defects hide. Do NOT reach it the wrong way — a test written only to
raise the number, asserting nothing an obligation requires, makes the suite slower without making
it stronger. Every test still traces to a fixture or an acceptance criterion; the 85% is a floor
those honest tests must clear, not a target to game.

## Never edit the implementation to make a test pass

If a test fails, one of three things is true, and they need opposite responses:

1. **The implementation is wrong** → report it. `CodeFixAgent` repairs implementations; you do
   not.
2. **The test is wrong** → fix the test, but only if you can point at the fixture or contract
   that shows what it should have been.
3. **The fixture and the implementation disagree because the spec was ambiguous** → a gap.
   Do not resolve it by changing either side.

Changing the implementation to match a test you just wrote is circular, and it converts a
real defect into a silent one.

## Interpreting a validation run

`50-validation/Validation-Result.json` is written by a script: commands, exit codes, output
tails. It reports; it does not judge. You judge.

Read `commandEvidence` on each check before concluding anything:

- A **declared** command failing → the code is wrong. Act on it.
- A **conventional** command failing → possibly the code, possibly the wrong command for this
  repository. Check whether the command makes sense here before sending the fixer after
  imaginary defects. A `mypy` failure in a project that never ran `mypy` is not a code defect.
- A check `SKIPPED` because the toolchain has no such command → not a finding at all. Python
  projects have no build step.
- **Exit code 137, or a timeout with no output** → the process was killed, usually for memory.
  That is an environment failure, not a test failure. Say so; a fixer told "the tests failed"
  will rewrite working code.

Distinguish these in your report. Most wasted retries in this pipeline come from a validator
reporting "tests failed" when the tests never ran.

## Your validation report

Write `40-codegen/Code-Validation.json`. In `payload`:

```json
{
  "verdict": "PASS|FAIL",
  "checks": [{"check": "test", "status": "FAIL", "declared": true, "interpretation": "code defect"}],
  "findings": [
    {
      "id": "CV-002",
      "severity": "HIGH",
      "file": "src/validation/message_validator.py",
      "line": 41,
      "problem": "SchemaError is raised without the message id; FIX-004 asserts it is present",
      "evidence": "pytest tests/test_validator.py::test_schema_error_carries_id — AssertionError: id not in error",
      "requirement": "FR-004",
      "fixTarget": "CoreImplementationAgent"
    }
  ],
  "environmentFailures": [],
  "retryRequired": true,
  "retryReason": "message id must be attached to SchemaError per FIX-004"
}
```

Quote the actual failure output in `evidence`. The fixer works from that text; a paraphrase
costs it a round of investigation, and there are only two rounds.

`fixTarget` names **one** agent per finding. That is what gets retried.

Your envelope `status` is `OK` when you completed the validation, even when `verdict` is
`FAIL`. Status describes whether you worked; verdict describes what you found.
