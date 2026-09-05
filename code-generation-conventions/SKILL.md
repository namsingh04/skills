---
name: "code-generation-conventions"
description: "Write code from an implementation spec that reads as though the repository's own team wrote it - in the discovered language, matching discovered patterns, reusing what exists, with no placeholder bodies, and writing every file the fileMap names. Use by every code-writing agent."
version: 21
created: "2026-08-20"
updated: "2026-09-05"
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

**The reference's shared utilities are PROFILED for their API — you IMPLEMENT them for THIS solution,
you do NOT copy them.** `Repo-Profile.json` records, for each shared-utility module (the modules under
the reference's `utilities/`, `common/`, `helpers/`, `lib/` or `core/` directory), its PUBLIC API — the
exact function/class names and signatures the reference exposes. That API is the CONTRACT: implement each
utility module in THIS project with those SAME public names and signatures, so every caller resolves by the
profiled name and no test breaks on a renamed symbol. But the BODIES are yours to write for this solution —
do NOT copy the reference's file contents, and do NOT carry across values that are the reference's (a
constants module gets THIS solution's constants, from the Solution-Model, never the reference's). Match the
reference's public names; generate everything behind them. Do not fork a parallel helper beside a profiled
one, and do not substitute a language's stdlib API where the profiled utility API is the house convention.

Two different things, and keep them apart: **follow the reference's LAYOUT PATTERN; build the SOLUTION's
architecture inside it.** From the reference you take the CONVENTION: the doubled `<name>/<name>/` layout,
one config file per environment where it has per-env config, the manifest/entry-point/descriptor/test-dir
shape, the naming style, and the config-indirection idiom — this is why the project looks like it sits
next to the reference. But the PACKAGES and MODULES are the SOLUTION's, from the spec's fileMap, which the
spec derived from the Solution-Model's components — they may be packages the reference does NOT have, and
you MUST create exactly the packages the fileMap names. You do NOT reproduce the reference's business or
utility modules (or its own domain packages) unless the fileMap names them because the solution uses them.
Write exactly the files the fileMap lists — every one, at its
`targetPath` — and no file it does not list; the fileMap, not the reference's inventory, is the file set.

**Follow the reference's NAMING STYLE, but the SET of packages is the solution's (from the fileMap).**
Name a package the reference's way (its casing, its granularity idiom — helpers grouped vs split) and use
the reference's dir name where the solution has the same concept (`utilities/`, `test/`). Do not invent a
gratuitously different taxonomy for something the reference already names — but DO add the packages the
solution's architecture requires that the reference lacks. The directory set = the fileMap's directories
(the solution's design in the reference's style), not a copy of the reference's own directories.

**Keep the project's OWN package directory named EXACTLY as the reference names it — do not
"normalise" it.** The reference lays a project out as `<root>/<name>/<name>/` and the deep dir is put
on the path so only its CHILDREN (`auth/`, `utilities/`, …) are imported — the deep dir itself is
never `import`ed by name, so its name is correct as-is and the CI/CD pipeline depends on that exact
path. This is not just the reference's habit: the standards profile's naming convention makes the
**project / lambda-function directory `kebab-case` (hyphens)** — `snake_case` is for Python FILES and
identifiers (`main.py`, `helper_utils.py`, `lambda_handler`), NEVER for the project directory. So do
NOT rename the package dir to a snake_case identifier (e.g. swapping hyphens for underscores in the
skeleton's project-dir name), and do not write some files under the skeleton-named dir and others under a
renamed one — that splits
the project into two half-trees that neither import nor validate. Use the skeleton's exact path for
every file you write.

**Folder structure comes from BOTH sources — take the UNION.** The set of directories and required
files is the `projectSkeleton` (the reference's exact layout) TOGETHER WITH the `Standards-Profile`'s
`folderStructure` / `requiredFiles` when a standards document is present. Where the standard requires a
file the reference does not carry (for example a `Dockerfile`), you still produce it — the standard
outranks the repository. Where the reference carries per-environment `setup/`+`properties/` the standard
does not enumerate, you still produce those. Neither source alone is the whole structure; the project
must satisfy both.

**Package shape — follow the reference's PATTERN, generate neither more nor less than the requirement.**
- **Business logic lives in a subfolder, not flat.** If the reference keeps its business logic in a
  domain subfolder (the skeleton records `businessLogicSubdir`), put THIS project's business modules
  (clients, transformers, parsers, validators, processors) in a subfolder too — named for this project's
  own domain, NEVER the reference project's own name, and no reference-distinctive token may appear in
  any path or identifier. Do not scatter business modules flat beside `main.py`, and do not invent a
  role-based taxonomy the reference does not use — mirror the reference's package set from the skeleton.
  Write each file at the `targetPath` the spec's fileMap gives it; a file whose fileMap path is inside a
  package must NOT be written flat at the package root.
- **Tests live WITH the package**, in the reference's test directory name from the skeleton (e.g. `test/`
  when the reference uses `test/`, not an invented `tests/`), at the same level as `main.py` — validation
  runs in the package root and will not collect tests placed elsewhere. **Mirror the source tree inside
  the test dir** the way the reference does: a module at `<package>/<sub>/<module>.py` is tested at
  `<testdir>/<package>/<sub>/test_<module>.py`, not a flat `<testdir>/test_<module>.py`, when the
  reference nests its tests.
- **Nothing extra, nothing missing.** Write exactly what the spec/requirement asks: no unrequested
  feature nobody asked for, no second helper that duplicates a utility the project already provides, and no
  module the project never imports. Equally, drop nothing the requirement needs. When you consume an
  external response, capture it to match the sample response's actual shape, not an assumed one.
- **Per-environment config — GENERATE per the solution, in the config SCHEMA the profile records, never
  copied.** The config key schema is taken by the authority chain solution → standards → reference:
  `Standards-Profile.json` `configSchema` when the standards define one, else `Repo-Profile.json`'s
  reference config SCHEMA per environment — the section headers, the exact key names, and the file format
  (whatever the authority actually uses). Produce each per-env config file in that SAME format with those
  SAME keys, but with the VALUES for THIS solution taken from the `Solution-Model` (endpoints, ARNs, env
  config, mock data) — never the reference's values, never a differently-shaped file you invented, and
  never hardcoded.
  The format is the reference's (profiled); the content is the solution's (generated).
- **A gap the reviewer RESOLVED is authoritative.** When the manager hands you a resolved answer from
  `30-gaps/Gap-Resolutions.json` (a config value, a queue ARN, a VPC/subnet id, a delivery guarantee),
  that value is the source of truth for the field it answers — apply it to the code/config and REPLACE any
  `TODO`/placeholder it resolves. A resolution the reviewer supplied must never be silently ignored.

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

**Per-environment config: same KEYS as the reference, VALUES for the project you are building.**
The reference's `properties/*.properties` and `setup/*_setup.json` (or whatever per-env config the
`projectSkeleton` carries) define the exact key schema and file shape the CI/CD pipeline consumes —
that schema is the convention, so produce the SAME keys, in the same files, for every environment the
reference has. READ the reference's config to learn what each key is, then set the VALUE for THIS
solution from the `Solution-Model`, the `Standards-Profile` and the `Repo-Profile` — the queue names,
endpoints, ARNs, roles and toggles this project needs. Never copy the reference's own values (they
belong to the reference's service), and never hardcode a value the inputs do not support — if a
required value is genuinely absent, raise it as a gap. Same keys as the reference; values for the
project being built.

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

- **The checkout path is DETERMINISTIC — read it, do not compute it.** It is the `checkout` field of
  `<proven dir>/_run/run-config.json` (written by the init step as `<runRoot>/src`, the absolute cloned-repo
  path). Use that value verbatim as the base for every file you write. Do NOT derive `<run-root>/src` from
  the proven dir yourself — the proven dir IS `<run-root>/workflow_output`, and computing "`<run-root>/src`"
  from it is exactly what produced `workflow_output/src`. The checkout is a SIBLING of `workflow_output`,
  never a child of it.
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
