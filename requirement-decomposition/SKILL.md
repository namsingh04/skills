---
name: "requirement-decomposition"
description: "Separate business, functional, non-functional and technical requirements cleanly, and write each as a discrete testable statement with its source. Use in the analysis stage by the business, functional, non-functional and technical requirement agents."
version: 1
created: "2026-08-20"
updated: "2026-08-20"
---

# Requirement decomposition

Four categories, four agents, one shared set of rules. You are responsible for exactly one
category. Everything you find that belongs to another category you **hand over rather than
keep** — record it in `crossCategory` so the agent who owns it can pick it up, and do not
write it as though it were yours.

## What to read — nobody else will tell you

**Read these four files before anything else**, at `<run root>/workflow_output/`:

| File | What it gives you |
|---|---|
| `00-inputs/Input-Manifest.json` | what was supplied, and what was not |
| `00-inputs/Solution-Model.json` | the solution design — highest authority for what to build |
| `00-inputs/Jira-Model.json` | issues and acceptance criteria — authority for what "done" means |
| `00-inputs/Additional-Instruction.json` | scope: what is in, what is narrowed, what is excluded |

The run root is the directory **above** your working directory, so these are normally at
`../workflow_output/00-inputs/`.

This list is here, in the skill, because **it may not reach you any other way.** When you run
as a sub-agent your instruction is a single sentence composed by your manager — it names what
to produce and where to write it, and on 2026-08-20 it named no inputs at all. Every analysis
agent that run returned PARTIAL for exactly that reason: told what to extract, never told what
to extract it *from*. If your instruction does not name your inputs, do not guess and do not
work from the instruction alone — the four files above are your inputs.

If a file is absent or unparseable, say so in `warnings` and work from what remains. Only a
missing Solution-Model **and** Jira-Model together is `BLOCKED`.

## Which category is yours

Your manager's message names it. If it does not, infer it from the output path you were given —
`Business.json`, `Functional.json`, `NonFunctional.json`, `Technical.json` — and say in
`warnings` that you inferred it.

## The four categories, and the test that separates them

| Category | Answers | Test that it belongs here |
|---|---|---|
| **Business** | *Why does this exist? What outcome does it serve?* | Remains true if every technology in the system were replaced. Names no system, service or file. |
| **Functional** | *What must the system do?* | Can be stated as an observable behaviour: given this input or event, the system does this. |
| **Non-functional** | *How well must it do it?* | Has a measurable threshold — latency, throughput, availability, retention, recovery time, cost ceiling, compliance obligation. |
| **Technical** | *What is fixed about how it is built?* | Constrains implementation and comes from a source with authority to constrain it: a named runtime, protocol, region, integration, encryption requirement. |

The dividing line that matters most: **a business requirement that names a technology is
misfiled**, and a functional requirement with a number in it is usually two requirements — a
behaviour and a threshold.

## Writing a requirement

One obligation per statement. A sentence containing "and" usually holds two; split it.

Each requirement:

```json
{
  "id": "FR-004",
  "statement": "When an ingest message fails schema validation, the system writes it to the dead-letter queue and does not acknowledge the source message.",
  "category": "functional",
  "source": "solution:§3.2 Ingest flow",
  "acceptanceEvidence": "jira:ABC-123 AC-3",
  "priority": "MUST|SHOULD|COULD",
  "measurable": true,
  "derivedFrom": []
}
```

**`source` is mandatory.** A requirement you cannot attribute to a document, a table row, a
diagram edge or an acceptance criterion is one you invented. Delete it or raise it as a gap —
it is not a requirement.

**`acceptanceEvidence`** points at what would settle whether it is met. Where the Jira
acceptance criteria cover it, cite them; that is what they are for. Where nothing covers it,
say so — an untested requirement is a real finding.

## Untestable wording is a gap, not a requirement

"fast", "robust", "user-friendly", "scalable", "secure", "as needed", "appropriately".

Each of these is a number somebody has not written down. Raise a gap asking for it, record
the requirement with `measurable: false`, and give a `proposedDefault` where the documents
support one. Do not supply the number yourself: a latency budget you invented becomes a test
assertion, and then a production alert.

## Do not invent, do not merge, do not resolve

**Do not invent.** Where the source is silent, the answer is a gap. A plausible requirement
is indistinguishable from a real one three stages later.

**Do not merge a contradiction.** Two sources that disagree produce a `CONTRADICTION` gap with
both readings recorded. A compromise between them is a third requirement that nobody wrote,
and it will be wrong in the way that is hardest to detect.

**Do not resolve priority conflicts.** If the solution document treats something as essential
and Jira does not mention it, record both facts. The authority chain decides, not you.

## Do not cross into design

You are describing **what is required**, never **how it will be built**. No file names, no
class names, no library choices, no resource shapes. That is the specification stage, and the
firewall between them is deliberate: a requirement that arrives pre-designed removes the one
place where a human can still object to the design.

The exception is the **technical** category, which exists precisely to record constraints that
*are* about the how — but only constraints the sources actually state, never ones you would
choose.

## Output

Write your category's file: `10-analysis/Business.json`, `Functional.json`,
`NonFunctional.json` or `Technical.json`. In `payload`:

```json
{
  "category": "functional",
  "requirements": [ … ],
  "crossCategory": [{"statement": "", "belongsTo": "non-functional", "source": ""}],
  "coverage": {"sourcesRead": [], "sectionsWithNoRequirements": []}
}
```

`sectionsWithNoRequirements` is worth filling in honestly. A section of the solution document
that produced nothing is either genuinely background or something you missed, and being able
to tell those apart later is worth the line.

## Status

`OK` — every source read, every requirement extracted and attributed.
`PARTIAL` — extracted with gaps raised for what was unclear or unmeasurable. Normal.
`BLOCKED` — the input manifest or the solution model was missing or unparseable. Not "the
requirements were vague" — vague requirements are exactly what `PARTIAL` and gaps are for.
