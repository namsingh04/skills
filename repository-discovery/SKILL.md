---
name: "repository-discovery"
description: "Profile an existing repository in any language - layout, module boundaries, naming, test placement, error handling and configuration patterns - with a file path as evidence for every claim. When the solution names a reference project, profile that project completely as the structure to reproduce. Use when analysing the target repository before generating code into it."
version: 13
created: "2026-08-20"
updated: "2026-09-03"
---

# Repository discovery

You are reporting what this repository **is**, not what it should be. Generated code has to
look like it belongs, and the only way to achieve that is to describe the existing code
precisely enough that a generator can imitate it.

Nothing here is language-specific. `Toolchain-Profile.json` in `50-validation/` already
records the language and its commands — read it first and let it tell you what you are
looking at.

## What to read, and what to write

**You profile the TARGET CODE REPOSITORY — the checkout you are standing in.** Not the
workflow's own output folder.

On 2026-08-20 this agent was handed the instruction *"profile the target repository layout,
especially whether `10-analysis/` already exists and any existing analysis artifacts"* — which
points at the run's output directory, not at any source code. It is not what this stage is
for. If your instruction reads like that, disregard the framing and profile the repository.

Read:

| Source | Why |
|---|---|
| the checkout itself | the thing you are profiling |
| `50-validation/Toolchain-Profile.json` | language and commands already detected; confirm and correct |

Write **`10-analysis/Repo-Profile.json`**, and update `50-validation/Toolchain-Profile.json`
in place with anything you corrected — the language, the commands, the real project root in a
monorepo. Nothing else corrects that file; if you skip it, the validation stage runs the wrong
commands in the wrong directory.

These are stated here because a sub-agent's instruction is one sentence written by its manager
and may name neither your inputs nor your output path. If yours does not, this is the contract.

## Every claim carries a path

An observation without a file path is an opinion, and opinions are not actionable by a code
generator. Every pattern you report names at least one file, ideally two or three, and the
count of places you saw it.

Two examples is a pattern. One example is an instance. Say which.

## What to profile

**Layout.** The directory tree to a useful depth, with a one-line purpose for each
significant directory — inferred from what is in it, not from its name.

**Module boundaries.** What imports what. Where the seams are. Which directories are leaves
and which are hubs. New code goes at a seam; new code in the middle of a hub is how a
codebase stops being modular.

**Naming.** File names, module names, class and function names, test names, constants. Report
the actual convention with examples, including where it is inconsistent — an inconsistent
codebase is a fact the generator needs, because "match the surrounding code" means something
different when the surroundings disagree.

**Entry points.** How the thing is started, invoked, or deployed. What is at the top of the
call graph.

**Error handling.** How failures are raised, wrapped, logged and surfaced. This is the pattern
most often ignored by generated code, and the one reviewers notice first.

**Configuration.** Where settings come from, how they are read, how secrets are handled,
whether there is a config object or scattered environment reads.

**Testing.** Where tests live, how they are named, what framework, what a typical test looks
like, whether fixtures or factories are used, how external calls are faked. Quote one
representative test in full — it is the single most useful artifact you can hand the test
author.

**Dependency conventions.** What is already available. A generator reaching for a new
dependency when an equivalent is already used is a review comment every time.

## Method

Work outside-in and cheaply. Do not read the whole repository.

1. Read the toolchain profile, then the manifest files and any README or contributing guide.
2. List the tree. Identify the source root and the test root.
3. Pick **three representative source files** — ideally the entry point, one leaf module, one
   module that talks to something external — and read them properly.
4. Pick **one representative test** and read it properly.
5. Sample rather than exhaust. Ten files read carefully beat two hundred skimmed, and the
   patterns repeat.

Use `command_line` for structure (`ls`, `find`, `git ls-files`, `wc -l`) before reading
anything. Cheap, and it tells you where to look. **Keep every listing scoped and bounded** —
target the candidate roots, not the whole tree, and pipe through `head` (e.g. `git ls-files | head
-200`). A full `ls -R` / unbounded `find` of a large monorepo dumps tens of thousands of paths
into your context, and that listing is then re-sent on every later turn — it is the single largest
avoidable token cost in this stage.

## Sibling projects are the runtime standard — record the skeleton

A standards document states conventions in prose. It cannot know this repository's exact
layout. The repository itself does — and in a monorepo the strongest evidence of "what a
project here must contain" is the **projects already sitting next to the one you are adding**.

On 2026-08-21 a run generated seven `.py` files into a new lambda and **no `requirements.txt`**,
because nothing bound the code stage to the repo's shape. The repository held ~40 sibling
lambdas, every one of them carrying its own `requirements.txt` on an identical path. That
inventory was the answer, and it was never recorded.

**The TARGET project name comes from `00-inputs/Solution-Model.json` `targetProject.name`, NEVER from a
folder you find in the repository.** Read it first. The `layout` you record is `<sourceRoot>/<targetProject.name>`
(the doubled `<name>/<name>/` where that is the repo's convention). A repository can contain a folder of
a DIFFERENT name that a previous run left behind — that folder is an unrelated SIBLING, a candidate to
model on, but it is NEVER the target. Confusing a stale sibling for the target makes every downstream
stage build into the wrong folder. If `targetProject.name` is empty, the target is unknown — raise a gap,
do not adopt a repo folder as the target.

When the target is a new project among siblings that share a layout, find the **nearest sibling**
— the existing project most structurally similar to the one being added (its name is NOT the target's) —
and record its file inventory as a `projectSkeleton`, retargeted to the target name. This makes "the
config is present" a checkable condition the later stages and the pre-commit gate enforce, in ANY
language, without hard-coding filenames.

```json
"projectSkeleton": {
  "modelledOn": "<top>/<sibling-project>/<sibling-project>/",
  "layout": "<top>/<name>/<name>/",
  "requiredFiles": ["<manifest, e.g. the sibling's dependency file>", "<entry point>"],
  "requiredDirs": ["<dirs every sibling has>"],
  "utilityApi": [
    {"module": "utilities/log_utils.py",
     "publicSymbols": ["print_info_logs(message)", "print_error_logs(message)"]}
  ],
  "configSchema": [
    {"file": "properties/<env>.properties", "format": "ini",
     "sections": {"ssm": ["oauth-token-url", "models-api-base-url", "stream-api-base-url"]}},
    {"file": "setup/<env>_setup.json", "format": "json-flat",
     "keys": ["name", "runtime", "role", "vpc-config", "handler", "code", "memory-size", "timeout",
              "source-arn", "environment"]}
  ],
  "evidence": ["<path to a sibling file proving the layout>"]
}
```

- `requiredFiles` is the STRUCTURAL skeleton only — the files that define a project's SHAPE and that
  every sibling of this kind carries: the manifest, the entry point, the per-environment config files
  (`properties/<env>`, `setup/<env>`), descriptors, and the test directory. It is NOT a list of the
  reference's business or utility MODULES. A shared utility (`log_utils.py`, `dynamo_utils.py`) is
  recorded in `utilityApi` (below) as an API convention — NOT in `requiredFiles` — because the code stage
  creates it only if a solution module needs it; forcing it into `requiredFiles` ships stale code (a
  `dynamo_utils.py` in a project that never uses DynamoDB). Two siblings sharing a STRUCTURAL file makes
  it a convention; a shared utility module is a pattern, not a required file.
- **`utilityApi` PROFILES the reference's shared-utility modules for their PUBLIC API only** — for each
  module under the reference's `utilities/`/`common/`/`helpers/`/`lib/`/`core/` dir, list its public
  function/class NAMES and signatures (read them from the file), NOT its bodies or values. This is the
  contract the code stage implements against so callers resolve by name — it lets the code be GENERATED
  for this solution instead of the reference's file being copied. Language-agnostic: list whatever public
  symbols the reference's language exposes.
- **`configSchema` PROFILES the per-environment config FORMAT only** — for each config file the reference
  carries (`properties/<env>.properties`, `setup/<env>_setup.json`, or whatever it uses), record the
  `format` (e.g. `ini`, `json-flat`), the section headers and the exact KEY names — never the reference's
  VALUES. The code stage generates each config file in this format with THIS solution's values. Read the
  keys/sections from the reference's actual files; do not paraphrase or invent them.
- Omit the whole `projectSkeleton` ONLY when there is neither a named reference NOR a sibling to
  model on (a single-project repo, or a genuinely empty one). When a reference IS named it is never
  optional — see below. An absent skeleton asserts nothing; a wrong one blocks a good run.

### A named reference project outranks the nearest sibling

Read `00-inputs/Solution-Model.json` first. If it names a REFERENCE project to model on — phrasing
like "modelled on", "follow its conventions", "open that project first and follow it rather than
inventing" — then **that named project is the one to profile, and it is authoritative over a
merely-similar sibling.** The author pointed at it deliberately; the new project is meant to mirror
it.

For a named reference, **open it and record its COMPLETE inventory** — every directory and every
file it actually contains, at their real repo-relative paths, with a path as evidence — into
`requiredDirs` and `requiredFiles`. **"Complete" is scoped to the reference project's OWN directory:
run ONE bounded listing of that folder (`find <reference-project-dir>` or
`git ls-files <reference-project-dir>`) plus the target location — NEVER a repo-wide walk, and NEVER
sibling projects.** You need the reference's own tree and where the new project goes, nothing else;
scanning the other projects in a large monorepo is what times a run out (a 54-project repo was walked
in full on 2026-08-31). Do NOT reduce it to "manifest + entry point": whatever the
reference carries (per-environment config files, deployment/descriptor directories, sample/event
directories, auth or secret-handling modules, and so on) is part of the structure the new project
must reproduce, so capture ALL of it. Also record, with evidence, HOW the reference solves the
recurring problems the spec will need — configuration/secret retrieval, external-call auth, retry,
logging — so a downstream gap for any of those can be answered from the reference instead of raised.
Name no specific folder, file, environment, or library here: enumerate exactly what the reference
has on disk, nothing assumed.

**Emitting the `projectSkeleton` OBJECT is MANDATORY when a reference is named — it is the single most
important thing you produce.** Profiling the reference in prose is not enough: you MUST fill
`requiredDirs` and `requiredFiles` with its actual paths. **List every file individually** — one
entry per file, so a reference with N per-environment config files yields N entries, NEVER one
"representative" file with the rest implied. The spec stage builds its `fileMap` by copying this list
one-to-one; a skeleton that is absent, or that collapses per-environment files into one, is exactly
why a run ships a single environment's config files instead of every environment's. If you described
the reference but left `requiredDirs`/`requiredFiles` empty, you have not finished the task.

**The two fields that MUST be set are `modelledOn` and `layout` — they are the anchor everything
downstream depends on.** `modelledOn` is the reference project's directory (repo-relative, e.g. the
sibling the solution doc says to model on); `layout` is where the NEW project goes (repo-relative,
the reference's shape with the new name). Identifying these two from the solution doc and the repo is
the one judgement only you can make — always set both. A deterministic step after you
(`10a_backfill_skeleton`) then lists `modelledOn` exhaustively on disk and fills
`requiredDirs`/`requiredFiles` with the target's paths, so **every** file the reference carries
(config dirs like `setup/`, per-environment files, descriptors, helpers) is captured with no omission
— provided you set `modelledOn` and `layout` correctly. Set them precisely; the enumeration is handled
for you.

**You already ran the listing — RECORD it too when you can.** The `find`/`git ls-files` output you
just read IS the skeleton; transcribe those exact paths into `requiredDirs`/`requiredFiles`. (If you
fill them, the deterministic step leaves your list untouched; if you cannot, it completes them from
`modelledOn` — but it can only do that when `modelledOn` and `layout` are set.) Leaving the field empty
after you have seen the tree is the recurring, run-breaking failure (2026-08-31: the sibling tree,
`setup/` dirs and `template.yaml` were all listed in the agent's own shell output, yet `projectSkeleton`
came back blank and the generated project had no `setup/` at all).

**Reproduce the reference's EXACT nesting — including a repeated project-name directory and files that
sit at a MIDDLE level.** Many monorepos place a project at `<top>/<name>/<name>/` (the name repeats)
while keeping `template.yaml` and a `setup/` directory one level up at `<top>/<name>/`. Record each
path at the real depth you observed it: `requiredDirs` gets both `<top>/<name>/setup/` and the code
dir `<top>/<name>/<name>/…`; `requiredFiles` gets `<top>/<name>/template.yaml`,
`<top>/<name>/setup/<env>_setup.json` (one per environment), and every file inside the doubled code
dir. Do not flatten the two levels into one and do not drop the repeated segment — the spec and the
validation both key off these exact paths. Hardcode no name; copy the shape you saw.

**A CREATE run — where the new project does not exist on disk yet — is precisely when this is
mandatory, not when it is excused.** You model `layout` on the nearest sibling's real path (with the
new project's name substituted) and set `modelledOn` to that sibling. "The new folder isn't there yet"
is never a reason to leave the skeleton empty; the sibling is there, and it is the answer.

## Structure mismatch is a finding, not a failure

The repository may not look like the solution document expects. It may be empty, or a
monorepo, or organised on a principle nobody documented.

Report it. `status: PARTIAL`, a gap describing the mismatch, and a factual description of
what is actually there. The specification stage adapts to the real layout — it cannot adapt
to a layout you wished into the profile.

An **empty or near-empty repository** is a legitimate and common case for a create run. Say
so plainly, report `PARTIAL`, and note that the toolchain profile is the only convention
available. Do not invent a layout and describe it as discovered.

## Output

Write `10-analysis/Repo-Profile.json`. In `payload`:

```json
{
  "language": "python",
  "isEmpty": false,
  "sourceRoots": ["src/"],
  "testRoots": ["tests/"],
  "layout": [{"path": "src/handlers/", "purpose": "one module per external operation", "fileCount": 7}],
  "patterns": [
    {
      "aspect": "error-handling",
      "observed": "every handler wraps external calls in a try/except that re-raises as IngestError with the original attached",
      "evidence": ["src/handlers/ingest.py:41", "src/handlers/replay.py:33"],
      "occurrences": 6,
      "confidence": "high"
    }
  ],
  "naming": {"modules": "snake_case", "tests": "test_<module>.py", "evidence": []},
  "entryPoints": [{"path": "", "kind": "", "note": ""}],
  "testing": {"framework": "", "layout": "", "representativeTest": "tests/test_ingest.py", "excerpt": ""},
  "dependencies": {"declared": [], "notablyUsed": []},
  "projectSkeleton": {"modelledOn": "", "layout": "", "requiredFiles": [], "requiredDirs": [], "evidence": []},
  "notInspected": ["docs/", "infra/"]
}
```

`projectSkeleton` is the layout later stages MUST reproduce — the nearest sibling project's
required files and directory shape, however nested it is (some monorepos repeat the project name,
e.g. `<top>/<name>/<name>/`). Getting it right is what prevents a generator from inventing a `src/`
the repo does not have.

`notInspected` keeps you honest. Sampling is correct; pretending you read everything is not.

## Verify before you return

Silent omission of the skeleton is the recurring failure of this stage: the agent profiles the
reference or the siblings in prose, then returns `projectSkeleton` with empty `requiredDirs` and
`requiredFiles`, and every later stage loses the layout. Before you write the file, run this check on
your own output:

1. Did `00-inputs/Solution-Model.json` name a REFERENCE project, OR does the repo hold siblings that
   share a layout? If **yes**, `projectSkeleton.requiredDirs` and `projectSkeleton.requiredFiles`
   **must be non-empty**, each entry a real repo-relative path you listed, with `evidence`. Empty
   fields here are not an acceptable return — go back and enumerate them from the listing you already
   ran.
2. Emitting `projectSkeleton` with empty `requiredDirs`/`requiredFiles` is allowed **only** when you
   also raise a gap stating there was neither a named reference nor a sibling to model on. No such
   gap and empty fields is a defect, not a profile.
3. Re-read the file and confirm it parses, carries the envelope, and that the skeleton you just
   verified is actually in `payload` — not lost between your reasoning and the written bytes.

## What you must not do

- Do not recommend improvements. Refactoring is not this stage and not this run.
- Do not report a pattern you inferred from a directory name. Open the files.
- Do not describe what the repository *should* contain given the requirements. That is the
  specification stage's job, and doing it here contaminates the profile with the design.
