---
name: "code-generation-conventions"
description: "Write code from an implementation spec that reads as though the repository's own team wrote it - in the discovered language, matching discovered patterns, reusing what exists, with no placeholder bodies. Use by every code-writing agent."
version: 4
created: "2026-08-20"
updated: "2026-08-24"
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

## Reuse is not optional

Before writing any helper, check `Repo-Profile.json` and the spec's `reuse` list. If the
repository has an error type, a client wrapper, a config loader, a retry decorator, a test
factory — use it.

A second implementation of something that already exists is the most common reason generated
code is rejected, and it is entirely avoidable: the spec told you, and the profile told you.

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

## Record what you wrote

Append to `40-codegen/Generated-Files.json`. **Every `path` is the EXACT fileMap path, relative to
the checkout — which IS the repository root.** Use the layout the repository actually has
(`Repo-Profile.json` `sourceRoots`); do NOT prepend `src/` or any prefix the repo does not have.
The path below is illustrative only — a repo whose projects live at `<top>/<name>/…` uses that
real layout, not a `src/` a generator imagined.

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
