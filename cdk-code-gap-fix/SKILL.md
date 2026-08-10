---
name: "cdk-code-gap-fix"
description: "Repair generated CDK code from a structured validation report: triage findings by class, apply the smallest correct edit for each, re-run the ladder, and stop within a fixed round budget. Never weakens a type, a check or a test to make a gate pass, and escalates what it cannot legitimately fix."
version: 1
created: "2026-08-05"
updated: "2026-08-05"
---

## When to Use

Use after a validation stage has produced a `CodeValidationReport` containing findings, to
repair the code before it is reviewed or published.

---

## The One Rule That Overrides Everything

**Never weaken a check to make it pass.**

Specifically, and without exception:

- do not widen or erase a type, and do not introduce an escape-hatch type to silence a
  type error;
- do not add a suppression comment, an ignore directive, or a lint exclusion;
- do not delete, skip, or loosen an assertion or a test;
- do not remove a validation call, a guard, or a required branch;
- do not relax a configuration requirement into an optional one with a default;
- do not delete the code that fails rather than fixing it.

A gate that passes because the check was removed is worse than a red build: the build told
the truth and now nothing does. If a finding cannot be fixed without weakening something,
**leave it unfixed** and escalate it.

---

## Re-run the Validator's Ladder, Verbatim

The validation report carries `resolvedLadder`: the exact command strings that stage already
ran successfully in this repository. Run **those**, character for character. Do not compose a
build, test or synth command of your own.

A run was lost to precisely this. The validator ran `jest --runInBand`, which completed; the
repair stage then invented a bare `npm test`, which maps to plain `jest`, which never exits
because CDK tests leave open handles. Nothing bounded it and the run stalled indefinitely.

If a command in the ladder lacks a timeout wrapper, add one — never remove one. A command that
exceeds its budget is a finding, never a hang.

---

## Triage

Classify every finding before editing anything, and fix in this order — earlier classes
routinely cause later ones, and fixing a compile error often clears a dozen downstream
findings at once.

1. **Compile / resolution errors** — a symbol that does not exist, a wrong import path, a
   signature mismatch, a missing dependency. These are usually a divergence between what
   the generator assumed and what the repository actually exports. Resolve toward the
   repository: it is the mechanical fact, and the compiler does not negotiate.
2. **Synthesis errors** — a construct rejecting its props, a circular dependency, a
   missing required property, an unresolved context lookup. Fix the construct usage, not
   the construct.
3. **Literal fidelity** — a name shortened, re-cased, or replaced by a generated pattern;
   a placeholder collapsed to `null`; a numeric setting that does not match the
   specification. Re-copy the value from the InfraSpec, and where the InfraSpec lost it,
   from the Solution Design text. Never re-derive it from a convention.
4. **Spec coverage** — an in-scope resource that was never emitted. Emit it, following the
   generation skill's rules.
5. **Scope violations** — something built that belongs to `deferredScope`. Remove it, and
   remove what only existed to support it.
6. **Standards conformance** — a required branch, guard, tag, or structure the standards
   file mandates and the code omits. Add it as the standards file states.
7. **Configuration fields** — a required field not declared, or validated with a helper
   that does not exist. Declare it, and validate it with a helper the repository really
   has.
8. **Non-blocking findings** — lint, formatting, test failures. Fix them last, and only if
   the round budget allows. Never at the cost of a blocking finding.

---

## Minimal Diff

Each fix is the smallest edit that resolves its finding.

- Change the line that is wrong, not the file around it.
- Do not reformat, reorder, or restructure code you are not fixing. A large diff hides the
  real change and makes the next round unreadable.
- Do not refactor, rename, or "improve" anything the findings did not name.
- Do not touch files outside the file plan. If a fix genuinely requires a new file, add it
  to the plan and record it.
- Preserve the branch: confirm HEAD is the target branch before the first edit, and refuse
  to write if it is not.

---

## Rounds

Work in rounds. Each round is: apply fixes → re-run the ladder → compare.

- The round budget is fixed and small. Exhaust it and stop; do not negotiate with yourself
  for one more.
- **Stop early if a round makes no progress.** If the blocking finding count did not fall,
  and the same findings return with the same messages, another identical round will not
  help. Stop and escalate.
- **Stop early if a round makes things worse.** If the blocking count rose, revert that
  round's edits and escalate with the report from before them.
- Never re-run the whole generation stage. This stage repairs; it does not regenerate.

---

## Escalation

Anything still failing when the budget is spent, plus anything you declined to fix because
fixing it would require weakening a check, becomes a gap for human review.

Each escalated item states: the finding, what was tried, why it could not be fixed
legitimately, and what a human would need to decide. Where the underlying cause is a value
no source states, it is a gap in the ordinary sense — a value the reviewer can supply or
defer.

Escalation is a normal outcome. A stage that reports two unfixed findings honestly is
worth more than one that reports zero because it suppressed them.

---

## Output

Return a `CodeGapFixReport`:

```json
{
  "modelType": "CodeGapFixReport",
  "branch": "<branch repaired>",
  "roundsUsed": 0,
  "roundBudget": 0,
  "stopReason": "clean | budget-exhausted | no-progress | regression | nothing-to-fix",
  "rounds": [
    {
      "round": 1,
      "findingsAtStart": { "blocking": 0, "major": 0, "minor": 0 },
      "fixesApplied": [
        { "findingId": "", "file": "", "class": "", "edit": "<one line>" }
      ],
      "findingsAtEnd": { "blocking": 0, "major": 0, "minor": 0 }
    }
  ],
  "declinedToFix": [
    { "findingId": "", "reason": "<why fixing it would weaken a check>" }
  ],
  "unresolvedFindings": [ ],
  "filesTouched": [ ],
  "unplannedFiles": [ ]
}
```

Plus the reserved envelope keys from the `workflow-status-contract` skill.

Status mapping:

- **`OK`** — no blocking findings remain.
- **`PARTIAL`** — blocking findings remain, or items were declined or escalated. This is
  the expected status when the budget runs out, and it is not a failure of the workflow.
- **`BLOCKED`** — there was no validation report or no code to repair.

---

## Guardrails

- Do not weaken a type, a check, an assertion, a guard, or a test.
- Do not suppress, ignore, or exclude a diagnostic.
- Do not delete failing code instead of fixing it.
- Do not regenerate the stage's output from scratch.
- Do not exceed the round budget or loop on a repeating finding.
- Do not write on the source branch or switch branches.
- Do not stage, commit, or push.
- Do not claim a finding is fixed without re-running the ladder that produced it.
- Do not narrate. Return the report.

---

## Verification

Before returning, verify:

1. HEAD is the target branch.
2. Every entry in `fixesApplied` names a real finding id from the validation report.
3. The final round's `findingsAtEnd` came from an actual ladder re-run, not an estimate.
4. `unresolvedFindings` plus fixed findings account for every finding in the input report.
5. Every item in `declinedToFix` states which check would have been weakened.
6. No suppression directive, ignore comment, or widened escape-hatch type was introduced —
   search the diff for them explicitly.
7. No test or assertion was deleted or skipped.
8. `filesTouched` is within the file plan, or the exception is in `unplannedFiles`.
9. `roundsUsed` does not exceed `roundBudget`.
10. The report parses as valid JSON with balanced braces and brackets.
