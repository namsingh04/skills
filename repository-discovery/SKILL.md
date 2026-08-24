---
name: "repository-discovery"
description: "Profile an existing repository in any language - layout, module boundaries, naming, test placement, error handling and configuration patterns - with a file path as evidence for every claim. Use when analysing the target repository before generating code into it."
version: 4
created: "2026-08-20"
updated: "2026-08-24"
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

When the target is a new project among siblings that share a layout, find the **nearest sibling**
— the existing project most structurally similar to the one being added — and record its file
inventory as a `projectSkeleton`. This makes "the config is present" a checkable condition the
later stages and the pre-commit gate enforce, in ANY language, without hard-coding filenames.

```json
"projectSkeleton": {
  "modelledOn": "<top>/<sibling-project>/<sibling-project>/",
  "layout": "<top>/<name>/<name>/",
  "requiredFiles": ["<manifest, e.g. the sibling's dependency file>", "<entry point>"],
  "requiredDirs": ["<dirs every sibling has>"],
  "evidence": ["<path to a sibling file proving the layout>"]
}
```

- `requiredFiles` are the files EVERY sibling has (by basename) — the manifest and entry point,
  not project-specific modules. Two siblings sharing a file makes it a convention; one does not.
- Omit the whole `projectSkeleton` when there is no sibling to model on (a single-project repo,
  or a genuinely empty one). An absent skeleton asserts nothing; a wrong one blocks a good run.

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

## What you must not do

- Do not recommend improvements. Refactoring is not this stage and not this run.
- Do not report a pattern you inferred from a directory name. Open the files.
- Do not describe what the repository *should* contain given the requirements. That is the
  specification stage's job, and doing it here contaminates the profile with the design.
