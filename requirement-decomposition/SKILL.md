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
