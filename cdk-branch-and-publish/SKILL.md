---
name: "cdk-branch-and-publish"
description: "Branch lifecycle and publication for a code generation workflow: check out the target branch or cut it from source before any code is written, then clean workflow artifacts out of the checkout, stage only planned repository source files, and commit with the issue key first. Never writes to the source branch and never force-pushes."
version: 1
created: "2026-08-05"
updated: "2026-08-05"
---

## When to Use

Two stages use this skill:

- **Branch setup** — immediately after the repository is cloned, before any stage reads or
  writes it.
- **Publish preparation** — after code generation, validation and repair are complete, to
  prepare and publish the commit.

---

# Part 1 — Branch Setup

## Objective

Leave the checkout on the **target branch**, so that every subsequent stage writes there.
The source branch is a baseline to read and to branch from; it is never written to.

## Procedure

1. **Resolve the target branch name.** Use the name supplied by the workflow input. If it
   is blank, take the branch naming rule from the standards file. Only if the standards
   file is silent, derive one from the issue key(s) driving the run. Record which of the
   three supplied the name.

2. **Refuse a same-name run.** If the resolved target equals the source branch, stop and
   return `BLOCKED`. Writing generated code onto the source branch is the single outcome
   this stage exists to prevent, and proceeding "carefully" is not an option.

3. **Fetch remote state**, pruning deleted refs, so the existence check below is accurate.

4. **If the target branch exists on the remote** — check it out and set it to track the
   remote branch. Record `createdFromSource: false`. An existing target branch is committed
   onto; earlier work on it is preserved.

5. **If it does not exist** — create it from the source branch's remote ref. Record
   `createdFromSource: true` and the source commit it was cut from.

6. **Verify.** Read the current branch back and confirm it equals the target. Confirm the
   working tree is clean. If either fails, return `BLOCKED` with what you actually found —
   do not proceed on the assumption that it worked.

## Output

```json
{
  "modelType": "BranchSetup",
  "sourceBranch": "",
  "targetBranch": "",
  "targetNameSource": "input | standards | derived",
  "createdFromSource": true,
  "baseSha": "<commit the branch starts from>",
  "headSha": "<current HEAD>",
  "remoteTracking": "",
  "workingTreeClean": true
}
```

Plus the reserved envelope keys from the `workflow-status-contract` skill.

---

# Part 2 — Publish Preparation

## Objective

Produce a commit that contains **repository source code and nothing else**, on the target
branch, with a message that leads with the issue key.

## Step 1 — Clean the checkout

The workflow itself creates files inside the checkout: installed dependencies, synthesis
output, caches, logs, temporary scripts. Synthesis ran as a **test**; its output must never
reach the branch.

Delete every such artifact before staging. Take the list of what the toolchain produces
from the repository's own ignore rules and its build configuration rather than assuming
fixed names — then verify by checking what is untracked after cleaning.

The workflow's own output directory lives **above** the checkout, at the run root, and must
never have been copied inside it. If a copy exists inside the checkout, delete it and
record a warning: something upstream resolved a path wrongly.

## Step 2 — Stage by allowlist

**Never stage everything.** Staging every change sweeps in whatever the workflow left
behind, and the whole point of Step 1 is undone by one broad add.

The allowlist is the code generation stage's **file plan**, intersected with what git
actually reports as changed. Stage exactly those paths, one by one.

- A file changed but not in the plan is **not staged**. Record it — it is either a stray
  write or a plan that was not kept current, and both matter.
- A dependency manifest or lockfile is staged only when the file plan declares a deliberate
  dependency addition. An incidental lockfile churn from an install step is not a change
  anyone asked for.
- A file in the plan that git reports as unchanged is recorded, not forced.

After staging, list what is staged and assert every entry is on the allowlist. If anything
else is there, unstage it before continuing.

## Step 3 — Compose the commit message

**The issue key comes first, then the description.**

```text
<KEY>[, <KEY>…]: <short description of the change>

<longer description>

Files changed:
  - <path>
  …

Spec: <InfraSpec reference>
```

Keys come from the workflow input, in the order given. Where the standards file prescribes
a commit message format, that format wins — but the key still leads, since that is the
stated requirement.

Never invent an issue key, and never omit one that was supplied.

## Step 4 — Commit and push

Commit the staged paths on the target branch. Push, setting upstream tracking the first
time the branch reaches the remote.

- **Never force-push**, under any circumstance, including a rejected push. A rejection
  means the remote moved; report it rather than overwriting someone's work.
- Never rewrite, amend, or squash existing history on the branch.
- Never push to the source branch.

## Step 5 — Hand off

Write a handoff record for the downstream pipeline that will run synthesis and deployment
after the change is merged: the branch, the commit, the stacks involved, and where the
specification lives. That pipeline should not have to re-derive any of it.

## Output

```json
{
  "modelType": "PublishPreparation",
  "branch": "",
  "commitSha": "",
  "commitMessage": "",
  "issueKeys": [ ],
  "artifactsRemoved": [ ],
  "allowlist": [ ],
  "staged": [ ],
  "rejectedFromStaging": [ { "path": "", "reason": "" } ],
  "plannedButUnchanged": [ ],
  "pushed": true,
  "remoteBranchCreated": false,
  "handoff": {
    "branch": "", "commitSha": "", "stacks": [ ], "infraSpecPath": ""
  }
}
```

Plus the reserved envelope keys from the `workflow-status-contract` skill.

---

## Guardrails

- Do not write, commit, or push to the source branch.
- Do not force-push, amend, squash, or rewrite history.
- Do not stage by wildcard or stage everything.
- Do not commit dependencies, build output, synthesis output, caches, logs, environment
  files, or the workflow's own output.
- Do not invent, reorder, or drop an issue key.
- Do not create a pull request, merge, tag, or deploy — none of those is this stage's job.
- Do not delete anything outside the checkout.
- Do not report `pushed: true` without confirming the remote accepted it.
- Do not narrate. Return the record.

---

## Verification

**Branch setup**, before returning:

1. The current branch equals the target branch.
2. The target branch differs from the source branch.
3. `createdFromSource` matches what actually happened, and `baseSha` is a real commit.
4. The working tree is clean.

**Publish**, before returning:

1. The current branch is still the target branch.
2. Nothing staged is outside the allowlist — list the staged paths and check each one.
3. No dependency directory, build output, synthesis output, cache, log, environment file,
   or workflow output path is staged.
4. The commit message's first line begins with the issue key(s) supplied.
5. Every supplied issue key appears in the message.
6. The commit exists on the target branch and its file list matches `staged`.
7. The push was accepted, and no force flag was used.
8. The source branch has no new commits attributable to this run.
9. The handoff record names a real commit and a real specification path.
10. The output parses as valid JSON with balanced braces and brackets.
