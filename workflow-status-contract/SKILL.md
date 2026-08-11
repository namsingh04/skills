---
name: "workflow-status-contract"
description: "The shared output envelope, upstream gating rules, source authority chain, and gap schema used by every agent in a multi-stage generation workflow. Guarantees that a stage degrades rather than fails, that a downstream stage can always decide whether to proceed or skip, and that no value is ever invented."
version: 1
created: "2026-08-05"
updated: "2026-08-05"
---

## When to Use

Use this skill in every agent node of a multi-stage workflow, alongside whatever
stage-specific skill the node already carries. It defines three things the other skills
assume but do not state:

1. The reserved keys every output file carries.
2. How a node decides whether to run, degrade, or skip, based on its upstream input.
3. Where a value comes from when several documents could supply it.

It defines no domain behaviour of its own.

---

## The Output Envelope

Every agent writes one JSON file and returns that same JSON as its answer. The file
carries the stage's own model keys **plus** these reserved keys as siblings — never as a
wrapper around the model. A downstream reader must still find the model's own top-level
keys at the top level.

| Key | Type | Meaning |
|---|---|---|
| `status` | string | `OK`, `PARTIAL`, `SKIPPED`, or `BLOCKED` |
| `stage` | string | This node's label, exactly as the workflow names it — never a skill name or a section heading from a skill |
| `runId` | string | The final path segment of the resolved RUN ROOT, which is already a unique per-run identifier. Read it; never generate one |
| `outputPath` | string | The absolute path this file was written to, read back from the real write |
| `upstreamStatus` | string | The `status` of the input actually read. `"N/A"` when the upstream is a form or a git operation and carries no envelope |
| `nextAction` | string | `PROCEED`, `PROCEED_WITH_GAPS`, `HUMAN_INPUT_REQUIRED`, or `SKIP_DOWNSTREAM` |
| `gaps` | array | Gap objects, schema below |
| `warnings` | array | `{ "code": "<slug>", "message": "<one line>", "sources": ["<source>"] }` |

### What each status means

- **`OK`** — the stage did its work and produced a usable model. Gaps may still be
  present; gaps alone never reduce the status.
- **`PARTIAL`** — the stage produced a usable model but something was degraded: an
  optional input was missing, a tool was unavailable, a human gate timed out, or a retry
  budget was exhausted. Downstream stages treat `PARTIAL` exactly like `OK` and read
  `warnings` to learn what was degraded.
- **`SKIPPED`** — this stage deliberately did no work because its upstream was `BLOCKED`
  or `SKIPPED`. The payload is empty.
- **`BLOCKED`** — this stage's *own* work could not be done at all.

### What `BLOCKED` is for

`BLOCKED` means: my input is missing, empty, or unparseable, or my own tool calls failed
beyond their retry budget, so I have nothing to produce.

`BLOCKED` is **not** for any of these, and a stage that returns it for one of them is
wrong:

- a gap count, however large;
- a severity judgement from a validator;
- a readiness flag from a downstream contract;
- an unresolved question or an unspecified value;
- a human reviewer choosing not to resolve something;
- a tool or binary being unavailable in the environment.

Unspecified values are the expected output of analysis, not a failure of it.

---

## The Run Folder

Everything in a run lives under one folder, `<root>`:

```
<root>/workflow_output/   every agent's output, and every input an earlier agent produced
<root>/src/               the working checkout — only generated code belongs here
```

**`<root>` is inherited, never re-derived.** Take it from the `outputPath` in your upstream
input: strip the filename and the trailing `/workflow_output`, and what remains is `<root>`.
The first agent in the workflow resolves it from a known starting point; every agent after it
copies that answer.

Do **not** work it out from `pwd`. That recipe only gives the right answer when the agent
happens to start in `<root>` or `<root>/src`, and when it does not, the result still *looks*
plausible — an absolute path one directory off, which nothing notices.

This is not hypothetical. Two agents in one run each resolved the folder for themselves and
landed one level apart. One wrote its model where the other was not looking; the manager found
the file missing and blocked the run. Every stage afterwards correctly wrote a `SKIPPED`
envelope, so the run cost real money, produced nothing, and reported success at every node.

Only when there is no readable upstream `outputPath` may you resolve it yourself, and then you
must record a warning so the discrepancy is visible.

## Cap What a Command Can Return, in Bytes

Tool output lands in your context. A single oversized result ends the turn with a context-limit
error, and there is no partial recovery — the work done up to that point is lost with it.

So every command that could return more than a screenful is **byte-capped**:

```sh
<command> 2>&1 | head -c 20000
```

**Do not cap by lines.** `head -n 200` and `sed -n '1,200p'` bound the number of lines, not the
amount of text, and a single line of a minified `.json` or bundled `.js` file can be megabytes
on its own. This is not a hypothetical distinction: a validation stage capped a recursive grep
at 220 lines, took **3.19 MB** back into its context, and died. Its retry did the same thing and
died the same way.

**Never search or list recursively from above your working directory.** A run root contains the
entire repository checkout, the installed skills, and every artifact the run has produced;
`grep -R` across it is what produced the 3.19 MB above. Root every search at the specific
directory you mean.

Best of all, do not search. If you were handed a path, open that path. Searching for a file you
already have the location of is how this failure starts.

## Never Hand-Escape a Large Document Into a Tool Argument

If your output is more than a few KB of JSON, do not pass it as the `content` argument of a
file-writing tool. Doing so means every quote inside the document must be escaped by hand, and
**one missed escape invalidates the entire tool call** — not just the document. The platform
rejects the call with *"Invalid input format. Expected an object with properties 'path' and
'content'"*, the arguments arrive as `null`, and there is nothing to correct because the
failure is in the envelope rather than in the content.

The failure mode is a loop: the agent regenerates the whole document, makes the same class of
mistake, and fails identically. One run escaped 21 keys correctly and missed two — the
document was several KB — and burned fifteen minutes across five attempts before it was
killed. Nothing in the error said which two quotes were wrong.

Instead:

1. Write a short script to a file at the run root — never inside the checkout.
2. Build the document as a native object and serialise it with a real JSON library
   (`json.dump(obj, f, ensure_ascii=True, indent=2)`). The library handles every quote,
   backslash, newline and non-ASCII character correctly; hand-escaping does not.
3. Run the script, then load the file back and confirm it parses.

No string concatenation, no f-strings, no heredocs, no manual quoting of the document. If you
find yourself typing a backslash before a quote inside your model, stop — that is the failure
beginning.

## Prove the File Exists Before Reporting It

After writing your output, list it — `ls -l <the exact path>` — and confirm it is there and
non-empty. Only then put that path in `outputPath`.

If the listing fails or the file is empty, the write did not happen: report the real error,
leave `outputPath` empty, and say plainly that the write failed. Never report a path you have
not just seen listed.

An agent once told its manager its report was at a path where no file existed. The manager
looked, found nothing, and blocked the run — a failure that presented as a missing model
rather than as the bad path it actually was.

---

## Never Report What You Did Not Run

Every field in the envelope is a claim, and a downstream stage cannot tell a measured value
from an invented one. So:

- If a tool call fails, report the **actual error text** and which tool produced it.
- Do not infer a cause for a failure, and never generalise one failed call into a claim
  about the environment. A delegation tool failing tells you nothing about whether git,
  python or the filesystem work. If you have not tried them, you do not know.
- Never write a status, path, SHA, branch, count or identifier you did not read from real
  output. An invented absolute path in `outputPath`, or an invented `runId`, is worse than
  an empty field, because everything downstream trusts it.
- If you genuinely could not run something, say exactly that: name the command, quote the
  error, and leave the field you could not determine as an empty string rather than a
  plausible-looking value.

This is not hypothetical. A node once called a delegation tool with an invented agent name,
got "not found", never invoked a shell at all, and then reported `BLOCKED` with the reason
*"agent environment could not run git commands"* — plus a fabricated output path and a
fabricated run id. Every part of that conclusion was invented to explain an unrelated
failure, and it would have sent a human debugging the wrong thing entirely.

**Tooling availability is a warning, not a blocker.** A missing binary or an unavailable
tool is recorded in `warnings`, and any gap raised for it carries
`blocksCodeGeneration: false` — it says nothing about whether code can be emitted.

---

## Upstream Gating

Reading the upstream `status` is the first thing a node does, before any other tool call.

1. **`OK` or `PARTIAL`** → proceed normally. `PARTIAL` is never a reason to stop.
2. **`BLOCKED` or `SKIPPED`** → do not fail and do not raise an error. Write your own
   output file with `status: "SKIPPED"`, `upstreamStatus` set to what you saw, an empty
   payload, and `nextAction: "SKIP_DOWNSTREAM"`. Return that as your answer.
3. **You cannot find, read, or parse the upstream result** → **fail open**. Proceed as if
   it were `OK`, and record a `warning` saying you could not read it.

Rule 3 matters more than it looks. Inability to *see* an upstream result is never evidence
that the upstream failed — the result may be on disk under a path you did not check, or
in a variable you were not given. A stage that blocks because it could not find its input
kills the run while everything upstream of it actually succeeded. Only positive evidence
of failure — an explicit error, or an empty or corrupt payload you actually read — is
grounds for anything other than proceeding.

The workflow must always reach its end node. A stage that returns nothing, loops forever,
or raises is worse than a stage that returns a `SKIPPED` or `PARTIAL` result a human can
read.

---

## Authority Chain

Every "where does this value come from?" question resolves in this order:

1. **The standards file** — the base. It governs conventions, policy, naming, structure,
   required commands, and required checks.
2. **The Solution Design** — governs what is being built: resource names, keys, settings,
   flows, and any environment or account values it states.
3. **The RepoProfile** — the repository as it actually is.
4. **A declared gap** — when no source states the value.

Rules:

- Take the value from the highest-ranked source that states it.
- **Never invent one.** A name, key, ARN, threshold, timeout, or identifier that appears
  in no source is a gap, and it stays a gap. Do not resolve a gap by guessing a plausible
  value to make a model look complete.
- When two sources disagree, use the higher-ranked one **and** record a `warnings` entry
  naming both. Never resolve a conflict silently.
- **One exception.** For mechanical facts required to compile — the symbol names, helper
  signatures, file paths, and import specifiers that actually exist in the checkout — the
  RepoProfile wins. The standards file describes policy; the compiler does not negotiate.
  Record the conflict as a warning either way, so the discrepancy stays visible.

A value stated by a source is never a gap. Before writing any gap, check the sources
above it in the chain: writing a value as both a requirement and a gap is a
self-contradiction, and it has happened.

---

## Gap Schema

Every gap is a structured object with exactly these keys. Never a free-text string, and
never with extra keys such as `severity`, `title`, or `impact`.

```json
{
  "id": "GAP-001",
  "field": "<the specific field or value that is missing>",
  "description": "<what is missing and why it matters downstream>",
  "source": "<which source was expected to state it>",
  "requiresHumanInput": true,
  "blocksCodeGeneration": false,
  "suggestedResolution": "<what a reviewer would most likely fill in, or an empty string>",
  "resolution": ""
}
```

`resolution` is always an empty string when a stage creates a gap. Only a human review
gate fills it in. A stage must never write its own guess into `resolution`.

### Setting `blocksCodeGeneration`

Set it `true` only when code genuinely cannot be produced **at all** without the value.

A value that is simply unknown at generation time becomes a **required configuration
field** — declared, validated at load, and supplied per environment. That is a normal,
correct, expected outcome and it is not a blocker.

Test yourself: if you cannot name the specific file or resource that could not be emitted
at all, the answer is `false`.

---

## Guardrails

- Do not wrap the stage's model inside a `payload` or `result` key. The reserved keys are
  siblings of the model's own keys.
- Do not omit the reserved keys because the stage succeeded. Every output carries all of
  them, every time.
- Do not return `status: "OK"` while claiming in prose that something failed. The status
  field is the contract; prose is not.
- Do not leave `warnings` empty when a conflict, a degradation, or a fail-open occurred.
  An empty `warnings` array is a claim that none did.
- Do not raise, exit non-zero, or return an empty answer as a way of signalling failure.
  Write the envelope and return it.
- Do not invent a value to avoid declaring a gap.

---

## Verification

Before returning, verify:

1. The answer parses as valid JSON with balanced braces and brackets, and no markdown
   fence around it.
2. All eight reserved keys are present at the top level.
3. `status` is one of the four permitted values.
4. `upstreamStatus` reflects what was actually read, or is recorded as unavailable with a
   matching warning.
5. `outputPath` is an absolute path that exists on disk, read back from the write you
   actually performed. It does not contain a placeholder such as `<ROOT>`, and it is not a
   path you assumed the environment uses.
5a. `stage` is this node's label, and `runId` is the final segment of the resolved RUN ROOT.
   Neither was invented.
6. Every gap is an object carrying exactly the eight gap keys, with `resolution` empty.
7. Every `blocksCodeGeneration: true` gap names a specific artifact that could not be
   emitted at all.
8. `warnings` names every conflict resolved through the authority chain and every
   fail-open taken.
