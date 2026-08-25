---
name: "active-directory-protocol"
description: "The swarm coordination protocol every agent obeys before doing any stage work: read the run's active-directory map first, act only on your own entry, read only the inputs it lists, skip if your output already exists, and write only your own file. Attached to every agent; makes parallel fan-out duplicate-free and in sync without any shared writes."
version: 1
created: "2026-08-24"
updated: "2026-08-24"
---

# Active-directory protocol

This workflow runs its agents as a swarm: many sub-agents fan out in parallel each stage. To make
that parallelism **duplicate-free and in sync on a file-based platform that has no locking**, there
is one shared map — the **active directory** — that everyone READS and nobody but the run's setup
script and the stage managers WRITE. Follow it before you do anything else.

## Read the directory first

Before any other work, read **`_run/active-directory.json`** (under the run root — normally
`../workflow_output/_run/active-directory.json` from the checkout). It is written once, at run
start, from the workflow graph. It lists every agent in the run and, for each:

```json
{
  "agent": "FunctionalRequirementAgent",
  "stage": "10-analysis",
  "outputPath": "10-analysis/Functional.json",
  "inputs": ["00-inputs/Input-Manifest.json", "00-inputs/Jira-Model.json"],
  "dependsOn": ["00-inputs"],
  "status": "pending"
}
```

Find **your own entry** (match on the agent name your manager addressed you by). It is the single
source of truth for what you read, what you write, and whether you need to run at all.

## Act only on your own entry

- **Read ONLY the paths listed in your entry's `inputs`.** They are exactly what your stage needs.
  Do not read the whole output tree, do not scan sibling stages, do not re-read files not listed —
  every extra file you open is re-sent to the model on every later turn and is the single largest
  avoidable token cost in this workflow. If your instruction names a file your entry does not list,
  prefer your entry; if you genuinely need one more, read that one, not everything.
- **Write ONLY your entry's `outputPath`.** One file, the one named. Never another agent's file —
  that is how parallel agents clobber each other, and it is unrecoverable. This is why nobody writes
  the active directory itself except setup and the managers.

## Skip if you are already done — no duplicate work

Before doing the work, check whether your `outputPath` already exists, parses as JSON, and has
top-level `status` of `OK` or `PARTIAL` (the completion rule from the run contract). If it does,
**you are DONE**: return that JSON unchanged and do not redo the work. This is what keeps a swarm
duplicate-free — the same agent is never run twice, whether because of a resume, a retry, or a
parallel manager re-delegation. `BLOCKED`, `SKIPPED`, unparseable, empty or absent → not done, do
the work.

## The directory is read-mostly

You never write `_run/active-directory.json`. It is created once by the setup script and its
`status` fields are reconciled only by your stage manager (a single writer) as agents return. If
your entry's `status` is already `OK`/`PARTIAL`, treat it as a strong hint you can skip — but the
authority is your output file on disk (per the skip rule above), because the file is what the next
stage actually reads. If the directory and the disk disagree, trust the disk and record a warning.

## Shared-workspace hygiene: a nested `src/src` is always wrong — delete it

The checkout is the repository root; on this platform it is itself a directory named `src`. A path
like `<checkout>/src/…` therefore means an agent resolved a path against the wrong root and created a
**doubled `src/src/`** — it is never a real repository layout. If you are a manager (or any agent)
and you observe a nested `src/src` directory inside the checkout, **delete the inner `src`** and move
its contents up one level to the correct path, then record the correction in `warnings`. Never
commit or hand downstream a `src/src` path. (Correct target paths come from `Repo-Profile.json`'s
`sourceRoots` and `projectSkeleton`; never prepend a `src/` the repository does not have.)

## If the directory is missing

On an older run, or if the setup step did not produce it, `_run/active-directory.json` may be
absent. **Fail open**: proceed exactly as your own instruction and the run contract tell you —
read the inputs your instruction names, apply the same skip rule against your own output file, and
write your one output. A missing directory is never a reason to stop; it only removes the map, not
the work.
