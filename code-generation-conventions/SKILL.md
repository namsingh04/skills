---
name: "code-generation-conventions"
description: "Write code from an implementation spec that reads as though the repository's own team wrote it - in the discovered language, matching discovered patterns, reusing what exists, with no placeholder bodies, and writing every file the fileMap names. Use by every code-writing agent."
version: 11
created: "2026-08-20"
updated: "2026-08-31"
---

# Code generation conventions

Your output is judged by one question: **could a reviewer tell your files from the existing
ones by style alone?** If yes, the work is not done, however correct the logic.

## What you read

`20-spec/Implementation-Spec.json` — what to build. This is your instruction set.
`20-spec/Integration.json` — the contracts.
`10-analysis/Repo-Profile.json` — how this codebase does things. **This is the standard** — its
conventions, naming, layout and `projectSkeleton` are the rules your code conforms to.
`50-validation/Toolchain-Profile.json` — the language and its commands.

**Read `20-spec/Implementation-Spec.json`'s `coverage.deferredToStage`.** Any entry naming
`codegen` is a requirement the specification stage deliberately handed to you — `TC-029:
requirements.txt must pin exact versions`, for instance. Treat those as your own obligations
and cite them in the `satisfies` of the files you write; nothing else will.

**You do not read the requirements models, and you must not.** If the spec does not say it,
it is not yours to infer. A gap in the spec is a gap to report, not a hole to fill with a
guess about what the business probably wanted — the specification stage exists precisely so
that those decisions are visible before they become code.

## Match the repository, then the language

The repository is the standard: match what `Repo-Profile.json` reports, and where it is
internally inconsistent, follow the convention that dominates and say so in your output. A
reviewer seeing one file in a new style needs to know it was deliberate.

Concretely, imitate:

- **Naming** — modules, functions, classes, constants, tests. Copy the observed convention,
  including its inconsistencies where the profile reports them.
- **Error handling** — the existing error types, the existing wrapping, the existing logging.
  Do not introduce a new exception hierarchy beside one that already works.
- **Configuration** — read settings the way the repository reads settings.
- **Comment density and style.** A codebase with no comments does not want yours; one that
  documents every public function wants that too. This is the tell reviewers notice first.
- **Imports and layout** — ordering, grouping, absolute versus relative.

## A standards profile, when present, outranks the repository

If `00-inputs/Standards-Profile.json` exists, read it first. Its `mandatory` rules are binding and
**override the repository's own conventions** where they conflict — the authority chain is
`solution doc > standards > jira > repository`, and the repository is the lowest. `advisory` rules
are preferences: follow them unless the repository clearly does otherwise. A standard that the spec
already recorded as overridden by the solution doc upstream is settled — follow the spec's
resolution, do not re-litigate it. Run the profile's `requiredCommands` (lint, coverage, type-check)
as part of "done". Any rule you break is a recorded `deviation` with its id. When the file is absent,
nothing here applies and the repository is the standard as usual.

## Reuse is not optional

Before writing any helper, check `Repo-Profile.json` and the spec's `reuse` list. If the
repository has an error type, a client wrapper, a config loader, a retry decorator, a test
factory — use it.

A second implementation of something that already exists is the most common reason generated
code is rejected, and it is entirely avoidable: the spec told you, and the profile told you.

## A reference project is a pattern, not a donor

When the spec names a REFERENCE project to model on — a sibling Lambda, an example service — study
its conventions and reproduce them in NEW files written for THIS spec's units. Do not copy its
source files across and rename them. Cloning ships the reference's business logic (and its bugs, and
dead code your spec never asked for), and a reviewer sees it immediately. Adopt the PATTERNS — the
layout, the naming, the config-indirection, the error handling — and write the behaviour your own
spec defines. Genuinely shared code is imported, never duplicated; but in a repo of self-contained
projects, reproduce the per-project helpers the convention calls for freshly, and test them to this
project's coverage floor rather than assuming the reference already did.

Two different things, and keep them apart: **reproduce the reference's STRUCTURE exactly; author the
CODE fresh.** The structure — the directory layout captured in `Repo-Profile.json` `projectSkeleton`,
one config file per environment where the reference has per-environment config, every descriptor and
sample directory it carries, and NO files it does not have (no invented modules, no extra package
markers) — is copied faithfully, because a reviewer expects this project to sit next to the reference
and look like it. The bodies inside those files are written for THIS spec. Collapsing several
per-environment files into one, or scattering the layout across a different shape, is as wrong as
cloning the logic — both make the project fail to match the reference it was told to follow. Name no
specific folder or file from your own knowledge; take the exact set from the profile.

**Use the reference's OWN directory names — do not invent your own taxonomy.** If the reference
groups its helpers under one directory, your helpers go in a directory of that same name; do not
split them into several new fine-grained subdirectories the reference does not have when it keeps
them together (or vice-versa). The set of directories in your project is the set in the
`projectSkeleton`, with the same names — no more, no fewer. A tidier structure you prefer is still
the wrong structure.

## No placeholders

Never write:

- `TODO`, `FIXME`, or `# implement me` in place of a body.
- A function that returns a hardcoded value where logic was specified.
- A `pass`, `NotImplementedError` or empty block presented as complete.
- A comment describing what the code would do if it were written.

If you cannot implement something — the spec is silent, a dependency is missing, a contract is
unresolved — **do not stub it and report success**. Report `PARTIAL`, name the unit in your
output as `notImplemented` with the reason, and let the validator and the summary carry it.
A stub reported as complete is worse than a missing file: the missing file fails loudly, the
stub passes and ships.

## Write the whole file

Produce complete, syntactically valid files. Not fragments, not diffs, not "the rest is
unchanged". If you are modifying an existing file, read it first, then write it back whole
with your changes integrated.

**Check `fileMap` for `create` versus `modify` before writing.** Writing a file marked
`modify` as though it were new overwrites existing work, and nothing in this run can recover
it.

**Write EVERY file the fileMap marks `create` — code AND config.** Deployment descriptors,
per-environment config files, sample/fixture files, property/settings files and manifests all count:
if the fileMap or the repo's `projectSkeleton` names it, produce it with real content. Creating an
empty directory and moving on is not "done" — an empty folder is never committed, and the run fails
the existence gate. A missing config file breaks a deploy exactly as a missing module breaks an
import.

## Record what you wrote

Record every file you wrote in YOUR OWN stage output — the `Core.json` / `Integration.json` /
`Tests.json` / `Scaffold.json` your manager named — under a `files` list in the shape below. A
deterministic step assembles `40-codegen/Generated-Files.json` from what is actually on disk after
the stage and merges your `unit`/`satisfies` onto it. Do **not** append to a shared
`Generated-Files.json` yourself: parallel agents appending to one file clobber each other, and the
manager's own report once overwrote the whole list, leaving validation with no file paths at all.
**Write every file INSIDE the checkout — and the checkout is the `src/` directory under the run
root.** The platform clones the target repository into `<run-root>/src/`, and the staging step lists
what to commit with `git` run from that directory, so only files under `src/` are part of the
repository at all. A file written anywhere else is orphaned: it is never staged, never committed, and
the validation run (which executes inside the checkout) cannot see or import it.

A `fileMap` path is **repo-relative** (for example `<top>/<name>/…`, the path as it will appear in the
repository). Resolve it against the checkout, so on disk each file lands at
`<run-root>/src/<fileMap path>` — e.g. `src/<top>/<name>/…`. The `src/` segment is the checkout
directory, not part of the repo layout; staging strips it, so the committed path is exactly the
fileMap path. Concretely:

- **The `src/` checkout is a SIBLING of `workflow_output`, not a child of it.** Both sit directly under
  the run root: `<run-root>/src/` (the repo) and `<run-root>/workflow_output/` (where your stage
  outputs go). Use the **absolute checkout path** you were handed. If all you have is the "proven dir",
  it ends in `/workflow_output`, so the checkout is its sibling — `<proven dir>/../src` — never a
  `src/` created *inside* `workflow_output`.
- **DO** write at `<checkout>/<fileMap path>` — join the repo-relative fileMap path onto the absolute
  checkout path (or `cd` into the checkout first).
- **DO NOT** write at `<run-root>/workflow_output/src/<fileMap path>`. That is the exact failure seen on
  2026-09-01: every file landed under `workflow_output/src/…`, which is excluded from staging and
  invisible to validation, so the run reported "files not in the checkout" and validation ran over the
  wrong tree. `workflow_output` holds stage JSON only — never source code.
- **DO NOT** write at the bare run root (`<run-root>/<fileMap path>`, outside `src/`) — also orphaned.
- **DO NOT** double the segment to `src/src/…` by adding `src/` to a fileMap path already resolved
  against the `src/` checkout.

Every code agent must resolve this the same way: **all files inside the `src/` checkout, once, at the
repo-relative fileMap path beneath it.**

```json
{
  "files": [
    {
      "path": "<exact fileMap path from the spec, relative to the repo root>",
      "action": "create",
      "unit": "SPEC-007",
      "satisfies": ["FR-004"],
      "linesWritten": 84,
      "reused": ["<repo-relative path>:IngestError"],
      "deviations": [{"from": "STD-012", "why": "", "approved": false}]
    }
  ],
  "notImplemented": [{"unit": "SPEC-011", "why": "contract CON-004 has no response shape; gap GAP-spec-009"}]
}
```

`satisfies` feeds the traceability matrix. A file that satisfies nothing is a file the spec
did not ask for.

`deviations` is for the times you had to break a rule. Say which, and why. An unrecorded
deviation looks like a mistake; a recorded one is a decision.

## Keep context cheap

You already have the spec, the repo profile and your fileMap — that is what to build. When you do
touch the filesystem, keep `ls`/`find`/`cat` scoped to the TARGET project directory and pipe
listings through `head`. Never `ls -R` or `find` the whole monorepo: a large listing lands in your
context and is re-sent on every later turn, which is the biggest avoidable cost in this stage.

## Verify before reporting

1. The file exists at the path you intended — list it.
2. It parses. Use the toolchain profile's own tools: a syntax check, an import, a compile.
   Syntactically broken output wastes a full validation round and one of two retries.
3. Every import you wrote resolves to something that exists — either in the repository, in the
   declared dependencies, or in a file another agent in this stage wrote. Do not import a
   module you assumed a sibling would create; check, or specify it yourself.
4. You wrote every unit assigned to you, or listed the ones you did not in `notImplemented`.
