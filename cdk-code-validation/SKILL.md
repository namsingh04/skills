---
name: "cdk-code-validation"
description: "Validate generated CDK code by discovering and running the repository's own build ladder up to synthesis, then checking the result against the InfraSpec and the standards file. Reports honest, structured findings and degrades gracefully when tooling is unavailable, never inventing a pass."
version: 1
created: "2026-08-05"
updated: "2026-08-05"
---

## When to Use

Use in the validation stage after CDK code has been generated or repaired, to establish
whether the code compiles, synthesises, and matches what was specified.

This is the only stage permitted to run builds. Discovery and design stages read source;
they do not build.

---

## Scope of the Claim

Synthesis is run here **as a test**. It proves the app is well-formed and the constructs
resolve. It is not a deployment, and deployment is not this workflow's job — a downstream
pipeline runs synthesis and deployment for real after the change is merged.

State only what was verified. If synthesis did not run, the code is not "deployable" and
must not be reported as such.

---

## Step 1 — Discover the Ladder

**Do not hardcode build commands.** Different repositories use different package managers,
scripts, and entry points. Resolve the ladder in this order:

1. **The standards file** — if it prescribes the commands or checks a change must pass,
   those are the ladder, in the order it gives.
2. **The repository's own scripts** — the declared scripts in its package manifest, and its
   CDK configuration for the app entry point and any required context keys.
3. **The toolchain the repository declares** — engine and runtime version constraints, and
   any lockfile that dictates the install command.

Record the resolved ladder before running it. If a step in the standards file names a
command the repository does not provide, that is a warning, not a failure.

Typical shape of a resolved ladder, in order:

1. dependency install (respecting the lockfile and declared engine version);
2. type-check / compile;
3. **synthesis, as a test**, with any context keys the authority chain supplies;
4. unit tests, if the repository declares them — **non-blocking**;
5. lint / format check, if the repository declares them — **non-blocking**;
6. any additional check the standards file requires, such as a policy or security linter,
   run only when the standards file or the repository declares it — **non-blocking**.

### The environment value

Synthesis usually needs an environment or context value. Take it from the standards file,
then the Solution Design, then the repository's own configuration. If **no** source states
one, that is a gap raised for human review — not a guess, and not a default such as `dev`.
Record synthesis as not-run with that reason, and continue.

---

## Step 2 — Run It

Run each step in order, capturing the command, its exit code, and its output.

- A failing **blocking** step (install, compile, synthesis) stops the ladder — later steps
  would only report noise. Record where you stopped.
- A failing **non-blocking** step is recorded and the ladder continues.
- A step whose tool is unavailable is recorded as `not-run` with the reason, and the ladder
  continues. Missing tooling is never reported as a pass.

Keep output bounded. Capture the errors and a tail of the log, not the whole build
transcript.

---

## Step 3 — Static Conformance

Independently of the build, check the emitted code against what was specified:

- **Spec coverage** — every resource the InfraSpec marked `CREATE` or `EXTEND` and placed
  in scope appears in the code. A missing one is a finding.
- **Scope discipline** — nothing in `deferredScope` was built. An extra one is a finding.
- **Literal fidelity** — every stack name, resource name, key name and numeric setting in
  the code matches the InfraSpec or Solution Design character for character. A name that
  has been shortened, re-cased, or replaced by a generated pattern is a finding.
- **Placeholder integrity** — no emitted name contains the string `null` where a value
  belongs, and none begins `null-`. Templated placeholder tokens appear as the source
  writes them.
- **No leftover markers** — no `TODO`, `FIXME`, or `CONFIG_REQUIRED`-style placeholder
  survives for a value the sources actually state.
- **Configuration fields** — every field the generator declared required is validated at
  load with a helper that exists in the repository.
- **Standards conformance** — the checks the standards file states, applied as it states
  them.
- **Acceptance criteria** — each criterion maps to at least one emitted or modified file,
  or is recorded as uncovered.

---

## Bounding Every Command

Wrap every install, build, test and synth command in a timeout, and record the wrapper as part
of the command. A command that exceeds its budget is a **finding** — report the command and the
limit — never a hang.

**Never invoke a bare test script.** A package manager's `test` script with no flags frequently
does not exit: Jest keeps the process alive when tests leave open handles, which CDK and AWS
SDK tests routinely do. A run stalled for over half an hour that way, on a suite that had
already finished. Use flags that force a non-interactive, self-exiting run — for Jest,
`--ci --runInBand --forceExit`, and the equivalent for whatever runner the repository declares.

**Do not pipe a long command through `tail`.** It suppresses all output until the command
exits, so a hang and a slow run look identical and there is nothing to report. Redirect to a
file and read the tail of the file afterwards.

`resolvedLadder` must record the **exact command strings that were run**, timeout wrapper and
flags included. The repair stage re-runs them verbatim, so anything paraphrased there becomes a
command nobody has tested.

---

## Findings

Every problem is one finding:

```json
{
  "id": "FIND-001",
  "category": "compile | synth | test | lint | spec-coverage | scope | literal-fidelity | placeholder | standards | acceptance-criteria | tooling",
  "severity": "blocking | major | minor",
  "file": "<repo-relative path, or empty>",
  "line": 0,
  "message": "<what is wrong, in one sentence>",
  "evidence": "<the compiler or tool text, trimmed>",
  "fixHint": "<what would resolve it>"
}
```

`blocking` means the ladder cannot pass with it present — a compile error, a synthesis
failure, a missing in-scope resource. `major` and `minor` are real problems that do not
stop the build.

---

## Output

Return a `CodeValidationReport`:

```json
{
  "modelType": "CodeValidationReport",
  "branch": "<branch validated>",
  "ladderSource": "standards | repository | mixed",
  "resolvedLadder": [ { "step": "", "command": "", "blocking": true } ],
  "toolingResults": [
    { "step": "", "command": "", "exitCode": 0, "ran": true, "reason": "" }
  ],
  "compileClean": true,
  "synthRan": true,
  "synthClean": true,
  "stacksSynthesized": [ "" ],
  "conformance": {
    "specCoverage": { "expected": 0, "found": 0, "missing": [] },
    "scopeViolations": [],
    "literalMismatches": [],
    "uncoveredAcceptanceCriteria": []
  },
  "findings": [ ],
  "summary": { "blocking": 0, "major": 0, "minor": 0 }
}
```

Plus the reserved envelope keys from the `workflow-status-contract` skill.

Status mapping:

- **`OK`** — every blocking step ran and passed, and there are no `blocking` findings.
- **`PARTIAL`** — the code compiles but something is degraded: a tool was unavailable, a
  non-blocking step failed, synthesis could not run for a recorded reason, or `major`
  findings remain.
- **`BLOCKED`** — this stage could not do its own work at all: there is no code to
  validate, or the checkout is unusable. A red build is **not** `BLOCKED` — it is a report
  with blocking findings, and the repair stage consumes it.

---

## Guardrails

- Do not hardcode a package manager, a script name, or a synthesis command.
- Do not invent an environment or context value to make synthesis run.
- Do not report a step as passing when it did not run.
- Do not modify code to make a check pass — this stage reports, it does not repair.
- Do not deploy, publish, push, or alter git state.
- Do not paste an entire build log into the report.
- Do not fail the workflow because the build is red. Report it.
- Do not narrate. Return the report.

---

## Verification

Before returning, verify:

1. `resolvedLadder` was recorded before execution, and `toolingResults` has one entry per
   ladder step.
2. Every step marked `ran: false` carries a reason.
3. `compileClean` and `synthClean` reflect real exit codes, not intent.
4. `synthRan: false` is accompanied by a recorded reason and, where the cause was a missing
   environment value, a gap.
5. Every blocking finding names a file, or explains why it has no file.
6. `summary` counts match `findings`.
7. Every InfraSpec in-scope resource appears in `specCoverage` as found or missing.
8. `status` is `OK` only when no blocking finding exists and every blocking ladder step
   passed.
9. The report parses as valid JSON with balanced braces and brackets.
