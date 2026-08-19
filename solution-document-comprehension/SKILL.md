---
name: solution-document-comprehension
description: Read a solution design document - prose, business architecture diagram, mermaid source, mock data, and tables of IAM policies, VPC settings and other configuration - and turn it into one normalized model. Use when ingesting the solution design from Confluence or an uploaded file.
---

# Solution document comprehension

The solution design is the **highest authority** in this workflow for what gets built. Your
job is to get everything out of it, in a form later stages can use without re-reading it.

It is a mixed document. Treat each part with the method it needs, then reconcile them.

## The parts, and what each is good for

| Part | What it establishes | Trap |
|---|---|---|
| Prose | Intent, context, constraints | Prose describing a diagram is a summary, not a source. Read the diagram. |
| Business architecture diagram | Components and their relationships | An image you cannot read is a gap, not an absence. Say you could not read it. |
| Mermaid source | The same relationships, machine-readable | Where mermaid and the rendered image differ, mermaid is the source — it is what the author last edited. |
| Tables (IAM, VPC, config) | Exact values, and the only place they appear | A table cell is authoritative and must be transcribed exactly, not paraphrased. |
| Mock / sample data | Request and response shapes, the basis of every test | Sample data is *evidence of a contract*, not decoration. It survives into `Test-Fixtures.json`. |

## Method

**1. Inventory before extraction.** List every section, table, diagram and code block, with
its heading. Record the inventory in your output. A document whose structure you cannot list
is one you will silently read half of, and half a solution document produces confidently
wrong code.

**2. Tables verbatim.** Use the structured table extraction skill. Preserve every column,
including ones that look irrelevant — a column you drop is a permission or a CIDR nobody can
recover later. Keep cell values exactly: `10.0.0.0/16` is not "the VPC range", and
`s3:GetObject` is not "read access to the bucket".

**3. Diagrams through mermaid where it exists.** Use the mermaid interpretation skill.
Extract nodes, edges, direction and labels as data. An arrow direction is a dependency
direction and later becomes a call direction; getting it backwards inverts the integration
design.

**4. Sample exchanges into fixtures.** Every request/response pair, example payload, or
sample record becomes a structured fixture with its operation name, direction, content type
and the exact body. These become the tests. Do not normalise, prettify or "fix" a sample —
if the sample is malformed, that is a gap worth raising, and possibly the real requirement.

**5. Reconcile.** Where the prose, the diagram and the tables disagree, record **all**
readings and raise a `CONTRADICTION` gap. Do not pick the one that seems most likely. On the
authority chain the solution document outranks everything else, so a contradiction *inside*
it has no tie-breaker above it — only a human can settle it.

## Output

Write `00-inputs/Solution-Model.json`. In `payload`:

```json
{
  "source": {"type": "confluence|file", "reference": "", "retrievedAt": "", "title": ""},
  "inventory": [{"heading": "", "kind": "prose|table|diagram|mermaid|sample", "ref": ""}],
  "components": [{"id": "", "name": "", "responsibility": "", "source": ""}],
  "relationships": [{"from": "", "to": "", "kind": "", "label": "", "source": "mermaid"}],
  "tables": [{"name": "", "kind": "iam|vpc|config|other", "columns": [], "rows": [], "source": ""}],
  "sampleExchanges": [{"operation": "", "direction": "request|response", "contentType": "", "body": {}, "source": ""}],
  "constraints": [{"statement": "", "source": ""}],
  "unreadable": [{"what": "", "why": ""}]
}
```

Every entry carries `source` — the heading or table it came from. Downstream stages cite
these when they trace a requirement to its origin, and a component with no source cannot be
traced.

## What you must not do

- **Do not interpret.** You are not deciding what to build, how to build it, or whether the
  design is sound. Extraction only.
- **Do not fill gaps.** A missing IAM policy, an unlabelled arrow, a sample response with no
  matching request — all are gaps. A plausible completion here becomes generated code that
  nobody chose.
- **Do not summarise tables.** Every row, every column.
- **Do not silently skip an unreadable part.** Put it in `unreadable` and raise a gap. "The
  diagram was an image I could not read" is useful; its absence looks like "there was no
  diagram", which is a different fact entirely.

## Status

`OK` — you extracted everything present and could read all of it.
`PARTIAL` — you extracted what you could and listed what you could not, with gaps raised.
This is a normal outcome for a real document.
`BLOCKED` — the document could not be retrieved at all, or was empty. Not "the document was
confusing".
