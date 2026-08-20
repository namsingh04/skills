---
name: "implementation-specification"
description: "Turn a validated requirements model into an implementation specification precise enough to code from - the single bridge between requirements and code, and the firewall that stops business requirements becoming implementation details directly. Use in the specification stage."
version: 1
created: "2026-08-20"
updated: "2026-08-20"
---

# Implementation specification

This stage is the **only** bridge between what was asked for and what gets written. Nothing
downstream reads requirements; everything downstream reads you.

That is a deliberate firewall. A business requirement translated straight into code skips the
one point where a human can still see the design and object to it — and the objection, when
it eventually comes, arrives as a rewrite instead of a comment.

## What you read, and what you must not

**Read:** `10-analysis/Functional.json`, `NonFunctional.json`, `Technical.json`,
`Repo-Profile.json`, `00-inputs/Standards-Profile.json`, `Solution-Model.json`,
`Additional-Instruction.json`, `50-validation/Toolchain-Profile.json`.

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
  "targetPath": "src/validation/message_validator.py",
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

**`targetPath` is a real path in this repository**, consistent with `Repo-Profile.json`. Not a
path you would have chosen for a greenfield project.

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
    "uncovered": [{"requirement": "NFR-005", "reason": ""}],
    "unitsWithoutRequirement": []
  },
  "assumptions": [{"statement": "", "because": "", "gap": "GAP-spec-004"}]
}
```

`fileMap` marks `create` versus `modify` explicitly. A code agent that modifies a file it
believed it was creating overwrites work, and that failure is not recoverable from within the
run.

`assumptions` records every default you proceeded on. Each one should have a gap id, and each
one appears in the run summary — this is how a reviewer finds the decisions nobody made.

## Status

`OK` — every MUST requirement covered, no blocking gaps.
`PARTIAL` — specified with recorded assumptions and gaps. Normal.
`BLOCKED` — the requirements model or repo profile was missing or unparseable. Not "the
requirements were incomplete" — incomplete requirements produce `PARTIAL` and gaps.
