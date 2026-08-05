---
name: "cdk-code-generation"
description: "Generate AWS CDK code into an existing repository from an InfraSpec: plan the files first, split the work across sub-agents by disjoint directory ownership, copy every literal from the specification verbatim, turn genuinely unknown values into validated configuration fields, and produce a file plan that doubles as the commit allowlist."
version: 1
created: "2026-08-05"
updated: "2026-08-05"
---

## When to Use

Use in the code generation stage, after an `InfraSpec` has been produced and any human gap
review has been merged, to write CDK code into an existing repository checkout.

Both the manager coordinating the stage and each code-writing sub-agent use this skill.

---

## Inputs

- `InfraSpec` (post-review, resolved) — what to build and how it fits the repository.
- The **standards file** — the base authority for conventions and structure.
- The **Solution Design** text — the authority for literal names and settings.
- `RepoProfile` — the repository as it actually is.

Apply the authority chain from the `workflow-status-contract` skill to every value:
standards file, then Solution Design, then RepoProfile, then a declared gap — with
RepoProfile winning on mechanical facts required to compile.

---

## Before Writing Anything: the Branch Check

Confirm the checkout is on the target branch before the first write, and refuse to write
if it is not. Generated code must never land on the source branch. If HEAD is not the
target branch, stop and return `BLOCKED` naming the branch you found — do not check out a
branch yourself, and do not write anyway.

---

## Step 1 — The File Plan

The manager writes the file plan **before** any sub-agent runs. It is the contract for the
whole stage and it is also the commit allowlist: nothing outside it may be staged later.

```json
{
  "filePlan": [
    {
      "path": "<repo-relative path>",
      "action": "create | modify",
      "owner": "<sub-agent that writes it>",
      "purpose": "<what this file contributes>",
      "specRefs": ["<InfraSpec resource or decision ids>"]
    }
  ]
}
```

Rules:

- Every file the stage intends to touch appears here. A file written but not planned is a
  defect, and it will not be committed.
- **Paths come from the authority chain, never from assumption.** Where the standards file
  prescribes a directory layout, use it; otherwise take the layout RepoProfile observed.
  Do not impose a conventional CDK layout on a repository that does not use one.
- Each file has exactly one owner. Two sub-agents must never own the same file.

---

## Step 2 — Disjoint Ownership

Sub-agents run in parallel, so their file sets must not overlap. Partition by role:

- **stack code** — stack definitions and the application entry point wiring;
- **construct code** — reusable constructs;
- **configuration** — per-environment configuration files and their loaders;
- **tests** — the test files for the above.

Each writes only files the plan assigns to it. **Cross-file wiring is the manager's job**,
after all sub-agents return: imports, stack instantiation, registration in the entry point,
and anything that requires knowing what another sub-agent produced. A sub-agent that edits
a file it does not own creates a lost update, because the parallel writer has the same file
open.

---

## Step 3 — Writing the Code

### Literals are copied, never regenerated

Every stack name, resource name, key name, ARN pattern, event pattern value, retention
period, timeout and retry count is copied **verbatim** from the InfraSpec — and where the
InfraSpec lost it, from the Solution Design text.

- Never shorten, abbreviate, expand or re-case a name.
- Never replace a stated name with a generated pattern or an expression that reconstructs
  it from configuration. If the design states the name, emit the name.
- **Templated placeholder tokens** — an environment or account placeholder written in the
  source — are reproduced byte for byte and resolved by the repository's existing
  configuration mechanism at deploy time. Never substitute a concrete value, and never let
  a placeholder collapse to the string `null`. **Any emitted name beginning `null-` is a
  bug**: go back to the specification and copy the real name.
- Never write a placeholder such as `CONFIG_REQUIRED` in place of a value the sources
  state. That marker is only for a value genuinely absent from every source.

### Unknown values become configuration fields

A value that no source states does **not** stop generation. Declare it as a required
configuration field:

- add it to the configuration shape the repository already uses;
- validate it where the repository validates its configuration, using the repository's own
  validation helpers — take their names from RepoProfile, because they must actually exist;
- fail at load time with a message naming the missing key, rather than defaulting silently.

This is a normal, correct outcome. It is not a gap that blocks code generation.

### Follow the repository, structurally

- Use the constructs, base classes, helpers and patterns RepoProfile found. Do not
  introduce a new architectural pattern, a new configuration mechanism, or a new directory
  convention.
- Extend an existing stack or construct where the InfraSpec classified the work `EXTEND`;
  create only what it classified `CREATE`; reference what it classified `REFERENCE`; touch
  nothing it classified `REUSE`.
- Where the standards file prescribes behaviour the repository does not yet exhibit —
  a required branch, a required guard, a required tag — emit it as the standards file
  states, and record the divergence from the repository as a warning.

### Scope

Emit only what the InfraSpec's `scope.inScope` covers. A resource in `deferredScope` is not
built, however convenient it would be and however much it shares a stack with something in
scope. Referencing an out-of-scope resource as context is fine; creating it is not.

---

## Step 4 — Report

Each sub-agent returns a `CodeGenReport`; the manager merges them.

```json
{
  "modelType": "CodeGenReport",
  "branch": "<branch HEAD was on while writing>",
  "filePlan": [ ],
  "filesWritten": [
    { "path": "", "action": "create|modify", "linesAdded": 0, "specRefs": [] }
  ],
  "configFieldsAdded": [
    { "key": "", "type": "", "validatedIn": "", "reason": "<which value was unknown>" }
  ],
  "literalsCopied": [ { "value": "", "from": "InfraSpec|SolutionDesign", "usedIn": "" } ],
  "deferredNotBuilt": [ { "item": "", "reason": "" } ],
  "divergences": [ { "topic": "", "standardsSays": "", "repoDoes": "", "chose": "" } ],
  "unplannedWrites": [ ]
}
```

Plus the reserved envelope keys from the `workflow-status-contract` skill.

---

## Guardrails

- Do not write on the source branch, and do not switch branches.
- Do not write a file that is not in the file plan. If one is genuinely needed, add it to
  the plan first and record it in `unplannedWrites`.
- Do not edit a file another sub-agent owns.
- Do not invent a resource, a name, a key, or a numeric setting.
- Do not build anything in `deferredScope`.
- Do not emit a name containing an unresolved `null`.
- Do not delete or rewrite existing repository code that the InfraSpec did not ask you to
  change.
- Do not run installs, builds, synth or deploys — validation is a separate stage.
- Do not commit, stage, or push. Publishing is a separate stage.
- Do not narrate. Return the report.

---

## Verification

Before returning, verify:

1. `git rev-parse --abbrev-ref HEAD` is the target branch, and was throughout.
2. Every path in `filesWritten` appears in `filePlan`, or in `unplannedWrites` with a
   reason.
3. No two sub-agents wrote the same path.
4. Every resource the InfraSpec marked `CREATE` or `EXTEND` and placed in scope has a
   corresponding file in `filesWritten`.
5. Every literal name in the emitted code matches the InfraSpec or Solution Design
   character for character — spot-check each stack and resource name.
6. No emitted name contains the string `null` where a value belongs, and none begins
   `null-`.
7. Every templated placeholder token appears exactly as the source writes it.
8. No `CONFIG_REQUIRED`-style marker survives for a value the sources actually state.
9. Every configuration field added is validated with a helper that RepoProfile confirms
   exists.
10. Nothing in `deferredScope` was built.
11. Every divergence between the standards file and the repository is recorded.
12. The report parses as valid JSON with balanced braces and brackets.
