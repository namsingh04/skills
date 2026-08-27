---
name: "implementation-specification"
description: "Turn a validated requirements model into an implementation specification precise enough to code from - the single bridge between requirements and code, and the firewall that stops business requirements becoming implementation details directly. Use in the specification stage."
version: 6
created: "2026-08-20"
updated: "2026-08-27"
---

# Implementation specification

This stage is the **only** bridge between what was asked for and what gets written. Nothing
downstream reads requirements; everything downstream reads you.

That is a deliberate firewall. A business requirement translated straight into code skips the
one point where a human can still see the design and object to it — and the objection, when
it eventually comes, arrives as a rewrite instead of a comment.

## What you read, and what you must not

**Read:** `10-analysis/Functional.json`, `NonFunctional.json`, `Technical.json`,
`Repo-Profile.json` (the repository IS the standard — its layout and conventions are what the
fileMap must match; use its `sourceRoots` exactly and never invent a `src/` prefix),
`Solution-Model.json`, `Additional-Instruction.json`, `50-validation/Toolchain-Profile.json`.

**Read but never translate directly:** `10-analysis/Business.json`. Business requirements
tell you *why*, and why is context for judgement, never a source of implementation detail. If
a piece of your spec traces only to a business requirement, you have skipped the functional
requirement that should sit between them — go and find it, or raise a gap saying it does not
exist.

## What a specification has to contain

Enough that an implementer never re-reads the source documents. If a code agent has to open
the solution design to know what to write, this stage has not finished.

For each unit of work:

```json
{
  "id": "SPEC-007",
  "unit": "ingest message validator",
  "targetPath": "<exact repo-relative path per Repo-Profile.json sourceRoots — NOT a greenfield src/>",
  "responsibility": "one sentence, and only one",
  "inputs": [{"name": "", "shape": "", "source": "20-spec/Integration.json#/contracts/2"}],
  "outputs": [{"name": "", "shape": ""}],
  "behaviour": [
    {"given": "a message missing the schema version header",
     "then": "raise SchemaError with the message id attached",
     "requirement": "FR-004"}
  ],
  "errorBehaviour": [{"condition": "", "response": "", "requirement": ""}],
  "dependencies": ["existing: src/errors.py IngestError"],
  "conventions": ["STD-012 test naming", "repo pattern: error-handling"],
  "satisfies": ["FR-004", "NFR-002"],
  "notes": ""
}
```

**`targetPath` is a real path in this repository**, taken from `Repo-Profile.json` `sourceRoots`
and `projectSkeleton` — not a path you would have chosen for a greenfield project. Use the repo's
real layout, however nested (some monorepos repeat the project name, `<top>/<name>/<name>/…`); do
NOT prepend a `src/` the repo does not have. A wrong prefix here becomes a doubled `src/src/` when
the code is written.

**`behaviour` is given/then, and every entry cites a requirement.** A behaviour with no
requirement id is a design decision you made unprompted — either find its requirement or drop
it. This citation is what makes the traceability matrix possible.

**`errorBehaviour` is not optional.** A unit specified only for the happy path will be
implemented only for the happy path.

## Reuse before invention

Read `Repo-Profile.json` before specifying anything new. If the repository already has an
error type, a config loader, a retry wrapper, an HTTP client — specify **using** it. A spec
that invents a second way to do something the codebase already does produces code that gets
rejected in review, and it is the most common failure of generated code.

State reuse explicitly in `dependencies`, prefixed `existing:`, with the path. Silence there
reads as "write a new one".

## Decisions you make, and decisions you escalate

**Yours:** which module something belongs in, how to decompose a requirement into units, what
to name things (per the conventions), the order of operations, which existing utility to
reuse.

**Not yours:** anything where the requirements genuinely do not say and the alternatives are
not equivalent — a delivery guarantee, a retention period, a permission boundary, whether a
failure retries or drops. Raise a gap. If you can proceed on a defensible default, set
`blocksCodeGeneration: false`, record the default in the spec, and let the review catch it.

The test: *if a reviewer would want to have been asked, ask.*

## Coverage

### A prohibition is discharged by absence, not by a unit

Some requirements say "must NOT". `TC-014: the DynamoDB retry mechanism must NOT be adopted`
cannot be cited in any unit's `satisfies` — there is no unit that implements not-doing-
something. Record it in **`constraintsObserved`**: the requirement id, what you deliberately
did not build, and what you used instead. That is the only form a prohibition can take, and it
is stronger traceability than a prose note, which a gate cannot count.

### A requirement another stage owns goes to that stage, by name

`TC-029: requirements.txt must pin exact versions` is a code-generation obligation. Record it
in **`deferredToStage`** with the stage named and why. The code stage reads that list as its
own input, so the requirement is carried rather than lost — and the coverage check stops
demanding that the spec satisfy something the spec cannot.

Neither of these is a way to make an inconvenient requirement disappear. Both require you to
say something explicit; silence is still an uncovered MUST.

Before you finish, check both directions:

- Every `MUST` requirement is cited by at least one spec unit. Requirements with no unit go in
  `uncovered` with a reason — and "no reason" means you missed one.
- Every spec unit cites at least one requirement. Units citing none are unprompted work;
  remove them or find the requirement.

Write the check into your output. It is the cheapest defect-finder in the pipeline.

## Output

Write `20-spec/Implementation-Spec.json`. In `payload`:

```json
{
  "targetLanguage": "python",
  "units": [ … ],
  "sharedTypes": [{"name": "", "shape": "", "usedBy": []}],
  "fileMap": [{"path": "", "purpose": "", "action": "create|modify"}],
  "reuse": [{"existing": "src/errors.py:IngestError", "usedBy": ["SPEC-007"]}],
  "coverage": {
    "requirementsCovered": ["FR-004"],
    "constraintsObserved": [
      {"requirement": "TC-014", "notBuilt": "no DynamoDB retry/requeue table",
       "insteadUsed": "SQS native redrive via maxReceiveCount"}
    ],
    "deferredToStage": [
      {"requirement": "TC-029", "stage": "codegen",
       "why": "requirements.txt is written by the scaffold agent, not specified here"}
    ],
    "uncovered": [{"requirement": "NFR-005", "reason": "", "gap": "GAP-spec-004"}],
    "unitsWithoutRequirement": []
  },
  "assumptions": [{"statement": "", "because": "", "gap": "GAP-spec-004"}]
}
```

`fileMap` marks `create` versus `modify` explicitly. A code agent that modifies a file it
believed it was creating overwrites work, and that failure is not recoverable from within the
run.

**The `fileMap` must list EVERY file the finished project needs — not only the behavioural
modules.** `Repo-Profile.json` `projectSkeleton` is the checklist, and when the solution names a
reference project the skeleton is that reference's COMPLETE inventory. Every `requiredDir` and
`requiredFile` it records — one entry per environment where the reference has per-environment config,
every descriptor/sample directory, every manifest, every helper the reference carries — becomes its
own `create` entry with a `purpose`. A code agent only writes what the fileMap names; a file the map
omits ships as an empty directory and fails the existence gate; per-environment files collapsed into
one, or a dir the reference has left empty, are the same failure. Take the exact set from the
profile — name no file from assumption. Do NOT add files the reference does not have.

**When `00-inputs/Standards-Profile.json` is present, it is an authoritative standards source.**
Apply the authority chain **solution doc > standards > jira > repository**: a `mandatory` rule in the
standards profile overrides jira and the repository's own convention, but a rule that contradicts the
solution design **loses** — record that as a gap, never silently apply it. Fold the surviving
mandatory rules into `units`, `fileMap` and `constraintsObserved` (cite the rule id); a mandatory
rule you cannot specify a way to meet is itself a gap. When the file is absent, the repository is the
standard exactly as before — this whole paragraph is a no-op.

`assumptions` records every default you proceeded on. Each one should have a gap id, and each
one appears in the run summary — this is how a reviewer finds the decisions nobody made.

## Status

`OK` — every MUST requirement covered, no blocking gaps.
`PARTIAL` — specified with recorded assumptions and gaps. Normal.
`BLOCKED` — the requirements model or repo profile was missing or unparseable. Not "the
requirements were incomplete" — incomplete requirements produce `PARTIAL` and gaps.
