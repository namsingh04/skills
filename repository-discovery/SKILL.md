---
name: repository-discovery
description: Profile an existing repository in any language - layout, module boundaries, naming, test placement, error handling and configuration patterns - with a file path as evidence for every claim. Use when analysing the target repository before generating code into it.
---

# Repository discovery

You are reporting what this repository **is**, not what it should be. Generated code has to
look like it belongs, and the only way to achieve that is to describe the existing code
precisely enough that a generator can imitate it.

Nothing here is language-specific. `Toolchain-Profile.json` in `50-validation/` already
records the language and its commands — read it first and let it tell you what you are
looking at.

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
anything. Cheap, and it tells you where to look.

## Structure mismatch is a finding, not a failure

The repository may not look like the solution document expects. It may be empty, or a
monorepo, or organised on a principle nobody documented.

Report it. `status: PARTIAL`, a gap describing the mismatch, and a factual description of
what is actually there. The specification stage adapts to the real layout — it cannot adapt
to a layout you wished into the profile.

An **empty or near-empty repository** is a legitimate and common case for a create run. Say
so plainly, report `PARTIAL`, and note that the standards document and the toolchain profile
are the only conventions available. Do not invent a layout and describe it as discovered.

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
  "conventionConflicts": [{"standardsRule": "STD-012", "repositoryDoes": "", "evidence": []}],
  "notInspected": ["docs/", "infra/"]
}
```

`conventionConflicts` is important: where the repository contradicts the standards document,
record both. **You do not resolve it** — the authority chain does, and it puts standards above
repository. But an existing codebase that consistently violates its own standard is evidence
worth surfacing, not a detail to smooth over.

`notInspected` keeps you honest. Sampling is correct; pretending you read everything is not.

## What you must not do

- Do not recommend improvements. Refactoring is not this stage and not this run.
- Do not report a pattern you inferred from a directory name. Open the files.
- Do not describe what the repository *should* contain given the requirements. That is the
  specification stage's job, and doing it here contaminates the profile with the design.
