---
name: "workflow-status-contract-core"
description: "The reserved output-envelope keys, the four status values and what each one means, the gap object schema, and how to set blocksCodeGeneration. The core reference for a multi-stage generation workflow: what a stage must emit so the next stage can decide whether to proceed, degrade, or skip. Verification here asks only whether the next stage has what it needs."
version: 1
created: "2026-08-12"
updated: "2026-08-12"
---

## When to Use

Use in every agent node of a multi-stage workflow, alongside whatever stage-specific skill
the node already carries.

**This skill is deliberately narrow.** It states the enumerations and schemas — the things
that must be identical across every stage and that a node cannot reason its way to. The
*procedures* that surround them — how to resolve the run folder, how to write a large
document, how to cap command output, how to prove a write, how upstream gating works, the
authority chain — are stated in each node's own instructions, which are present from the
node's first turn rather than arriving with a tool call. Where both describe the same rule,
the node's instructions are authoritative; this file is the reference for the shapes.

It defines no domain behaviour of its own.

---

## The Output Envelope

Every stage writes one JSON file and its answer carries the same reserved keys. They are
siblings of the stage's own model keys — **never** a wrapper around them. A downstream
reader must still find the model's own top-level keys at the top level.

| Key | Type | Meaning |
|---|---|---|
| `status` | string | `OK`, `PARTIAL`, `SKIPPED`, or `BLOCKED` |
| `stage` | string | This node's label, exactly as the workflow names it — never a skill name or a heading from a skill |
| `runId` | string | The final path segment of the resolved run root, which is already unique per run. Read it; never generate one |
| `outputPath` | string | The absolute path the file was written to, read back from the real write |
| `upstreamStatus` | string | The `status` of the input actually read. `"N/A"` when the upstream is a form or a git operation and carries no envelope |
| `nextAction` | string | `PROCEED`, `PROCEED_WITH_GAPS`, `HUMAN_INPUT_REQUIRED`, or `SKIP_DOWNSTREAM` |
| `gaps` | array | Gap objects, schema below |
| `warnings` | array | `{ "code": "<slug>", "message": "<one line>", "sources": ["<source>"] }` |

### What each status means

- **`OK`** — the stage did its work and produced a usable model. Gaps may still be present;
  gaps alone never reduce the status.
- **`PARTIAL`** — the stage produced a usable model but something was degraded: an optional
  input was missing, a tool was unavailable, a human gate timed out, or a retry budget was
  exhausted. Downstream stages treat `PARTIAL` exactly like `OK` and read `warnings` to
  learn what was degraded.
- **`SKIPPED`** — this stage deliberately did no work because its upstream was `BLOCKED` or
  `SKIPPED`. The payload is empty.
- **`BLOCKED`** — this stage's *own* work could not be done at all.

### What `BLOCKED` is for

`BLOCKED` means: my input is missing, empty or unparseable, or my own tool calls failed
beyond their retry budget, so I have nothing to produce.

It is **not** for any of these, and a stage that returns it for one of them is wrong:

- gaps exist, or many gaps exist — gaps are a normal output;
- a downstream stage might not be able to proceed — that is the downstream stage's call;
- an upstream envelope could not be read — proceed and record a warning instead;
- the work was hard, ambiguous, or incomplete — that is `PARTIAL`.

---

## Gap Schema

Every gap is a structured object with exactly these keys. Never a free-text string, never
an extra key such as `severity`, `title` or `impact`.

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

`resolution` is **always** an empty string when a stage creates a gap. Only a human review
gate fills it in, and a stage that writes its own guess there has that guess accepted as if
a person had made the decision.

A gap belongs in the `gaps` array and nowhere else. Three things must never appear inside
the stage's own model, because each has happened and each corrupted the output:

- gap text used as a value — a resource named `SNS topic GAP-004` is a gap wearing a name's
  clothing, and the code generator emits it literally;
- a gap described in prose in one of the model's own fields instead of listed as an object —
  a gate counts gap objects, so prose is invisible to it and the gap is silently lost;
- a gap the stage resolved itself — if a source states the value it was never a gap, and if
  no source states it the stage does not have the authority to decide.

### Setting `blocksCodeGeneration`

Set it `true` only when code genuinely cannot be produced **at all** without the value.

A value that is simply unknown at generation time becomes a **required configuration field**
— declared, validated at load, supplied per environment. That is a normal, correct, expected
outcome and it is not a blocker.

Test yourself: if you cannot name the specific file or resource that could not be emitted at
all, the answer is `false`.

---

## Guardrails

- Do not wrap the stage's model inside a `payload` or `result` key. The reserved keys are
  siblings of the model's own keys.
- Do not omit the reserved keys because the stage succeeded. Every output carries all of
  them, every time.
- Do not return `status: "OK"` while claiming in prose that something failed. The status
  field is the contract; prose is not.
- Do not leave `warnings` empty when a conflict, a degradation, or a fail-open occurred. An
  empty `warnings` array is a claim that none did.
- Do not raise, exit non-zero, or return an empty answer as a way of signalling failure.
  Write the envelope and return it.
- Do not invent a value to avoid declaring a gap.

---

## Verification

**Verify one thing: that the next stage has what it needs.** This is not a quality review of
the stage's own work — the stage-specific skill governs that. It is the handover check, and
every item below is something a downstream stage reads and would be broken without.

1. The answer parses as valid JSON, balanced braces and brackets, no markdown fence.
2. All eight reserved keys are present at the top level, as siblings of the model's keys.
3. `status` is one of the four permitted values, and `nextAction` one of the four permitted
   values.
4. `outputPath` is an absolute path that exists on disk and is non-empty, read back from the
   write actually performed — not assumed, not a placeholder such as `<ROOT>`, and never a
   path containing the repository checkout when the file belongs in the run folder.
5. `stage` is this node's label and `runId` is the final segment of the resolved run root.
   Neither was invented.
6. `upstreamStatus` is what was actually read, or is recorded as unavailable with a matching
   warning.
7. Every gap is an object carrying exactly the eight gap keys, with `resolution` an empty
   string — a gate reads this array and cannot see a gap written any other way.
8. Every `blocksCodeGeneration: true` gap names a specific artifact that could not be emitted
   at all.
