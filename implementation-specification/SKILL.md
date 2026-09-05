---
name: "implementation-specification"
description: "Turn a validated requirements model into an implementation specification precise enough to code from - the single bridge between requirements and code, and the firewall that stops business requirements becoming implementation details directly. Use in the specification stage."
version: 12
created: "2026-08-20"
updated: "2026-09-05"
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

**Writing a large spec file — build it in chunks, do not one-shot a huge `write_file`.** A full
design model (`BusinessLogic.json`, `Infrastructure.json`, `Integration.json`, or this
`Implementation-Spec.json`) is often tens of kilobytes, and a single `write_file` whose `content`
argument is that large intermittently malforms — the tool rejects it with `Invalid input format` and
the agent stalls retrying. This is exactly what stalled `BusinessLogicDesignAgent` on 2026-09-01 once
the project skeleton grew: the *content* was correct, only the single-shot write failed. So for any
sizable output, write it with `command_line`, in SEQUENTIAL APPENDED CHUNKS, to the SAME single file —
`cat > '<abs path>/File.json' <<'EOF' … first part … EOF`, then `cat >> '<abs path>/File.json'
<<'EOF' … next part … EOF` for each further section, keeping each chunk to roughly a screenful. The
file stays one file at its normal path — nothing downstream changes — it simply lands reliably instead
of intermittently. Then read it back to confirm it parses. (See `workflow-run-contract`.) Do not split
the model across multiple files, and do not trim its content to fit one call.

`fileMap` marks `create` versus `modify` explicitly. A code agent that modifies a file it
believed it was creating overwrites work, and that failure is not recoverable from within the
run.

**The `fileMap` has two kinds of entries — and mixing them up is the "stale reference code" defect.**
1. **STRUCTURAL CONVENTION files — reproduce from the skeleton.** The manifest, the entry point, one
   per-environment config file per environment the reference has (`properties/<env>`, `setup/<env>`),
   every descriptor/sample directory, the test directory — the files EVERY sibling of this shape carries.
   These come from `projectSkeleton` one-to-one: if the reference has per-env config across four
   environments, the fileMap has four entries, never one "representative". These are the project's SHAPE.
2. **BUSINESS + UTILITY modules — DESIGN from the SOLUTION, not the reference.** Every other module is
   derived from the Solution-Model's components (component + responsibility + relationships → a module),
   placed in the packages the SOLUTION's own architecture names — which may be packages the reference does
   NOT have. **The package division is `projectSkeleton.solutionPackages` when present** — the deterministic
   derivation records it there from the solution (its stated structure / components) by the authority chain
   solution → standards → reference; CREATE exactly those packages and place each component's module in its
   package. When it is absent, read the division from the solution yourself: one package per component/domain
   the solution describes, named as the solution names it (never a package name copied from the reference or
   invented).
   **A reference business/utility module is NOT the project's file: spec it ONLY if a solution module
   actually needs it.** The reference's utility *set* is a naming/API CONVENTION (if the solution needs a
   shared helper the reference also has, match the reference's name/API for it), never a file list to copy.

**Every fileMap entry that is not a structural convention file MUST trace to a solution
component/requirement.** A module that cites no requirement — a reference utility the solution never uses
— is stale by construction and must NOT be in the fileMap. This is the same coverage rule as your
units: no unit (and no file) without a requirement. Do not "copy the reference's complete inventory"; copy
its convention files and DESIGN the rest. If `projectSkeleton` is missing but a reference is named,
enumerate the reference's per-env config files yourself first (one environment standing in for all is a
defect), but still design the business modules from the solution.

**When `00-inputs/Standards-Profile.json` is present, it is an authoritative standards source.**
Apply the authority chain **solution doc > standards > jira > repository**: a `mandatory` rule in the
standards profile overrides jira and the repository's own convention, but a rule that contradicts the
solution design **loses** — record that as a gap, never silently apply it. Fold the surviving
mandatory rules into `units`, `fileMap` and `constraintsObserved` (cite the rule id); a mandatory
rule you cannot specify a way to meet is itself a gap. When the file is absent, the repository is the
standard exactly as before — this whole paragraph is a no-op.

**Pin the PROFILED contract so the code stage generates, never copies.** `Repo-Profile.json`
`projectSkeleton` now carries two profiles you must thread into the spec so the code agents match the
reference's shape without copying its content:
- **`utilityApi`** — for each shared-utility module, the reference's public function/class names +
  signatures. The unit for each utility file states that its public API MUST match those names/signatures
  (so callers resolve), while its BODY is implemented for this solution. Never specify "reuse the
  reference's file" — the body is generated.
- **`configSchema`** — for each per-environment config file the fileMap lists
  (`properties/<env>.properties`, `setup/<env>_setup.json`, …), the unit MUST state that file's SCHEMA
  from the profile: the `format` (e.g. `ini`, `json-flat`), the section headers, and the exact key names
  — and that the VALUES come from the `Solution-Model` for this environment. So the code stage generates
  config in the reference's format with the solution's values, not an AWS-SDK/`UPPER_SNAKE`/flat shape it
  invented, and never the reference's own values.

**Place every unit in its package — never flat.** Each unit's `targetPath` puts it inside the
`projectSkeleton` package it belongs to — the reference's domain package for business/client/transform/
parse logic, `utilities/` for shared helpers, `auth/` for auth, the reference's test dir name for tests
— NEVER flat beside `main.py`, and never an invented `clients/`/`transformers/` taxonomy the reference
does not have. The folder structure and file set in the fileMap are the UNION of the `projectSkeleton`
and the standards `folderStructure`/`requiredFiles`. Take keys, packages and structure from the profiles;
do not hardcode them. The standards document governs code CONVENTIONS (naming, error handling, coverage);
the reference supplies STRUCTURE and FORMAT; the solution supplies VALUES and behaviour.

`assumptions` records every default you proceeded on. Each one should have a gap id, and each
one appears in the run summary — this is how a reviewer finds the decisions nobody made.

## Status

`OK` — every MUST requirement covered, no blocking gaps.
`PARTIAL` — specified with recorded assumptions and gaps. Normal.
`BLOCKED` — the requirements model or repo profile was missing or unparseable. Not "the
requirements were incomplete" — incomplete requirements produce `PARTIAL` and gaps.
