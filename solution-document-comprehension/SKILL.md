---
name: "solution-document-comprehension"
description: "Read a solution design document - prose, business architecture diagram, mermaid source, mock data, and tables of IAM policies, VPC settings and other configuration - and turn it into one normalized model. Use when ingesting the solution design from Confluence or an uploaded file."
version: 1
created: "2026-08-20"
updated: "2026-08-20"
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

## When the document is a Confluence page

This is the usual case, and the tooling is narrower than you might assume. **One tool exists:
`confluence_get_page`.** There is no attachment download, no child-page listing, no search.
Everything below follows from that.

**Call it with `convert_to_markdown: true`.** Tables arrive as markdown tables, which is what
the table extraction skill expects. Raw HTML forces you to parse storage format by hand and is
where transcription errors come from.

**A page id or a URL both work.** From a URL, the id is the numeric segment after `/pages/`.
If you were given a title instead, it needs the space key alongside it.

**You get one page. You do not get its children.** If the page links to, or names, further
pages — "see the Integration Design page", a child-page macro, a table of contents pointing
elsewhere — **that content has not been retrieved.** Fetch each referenced page id you can
identify, and for any you cannot resolve, record it in `unreadable` and raise a `MISSING` gap
naming the link. A solution design split across a parent and four children, read as one page,
is the most expensive silent failure available here: everything downstream inherits the
missing four and nothing reports a problem.

**With `include_metadata: true` the page body arrives at `metadata.content.value`** — not at a
top-level `content` key. Look there before concluding the page was empty.

### Macros are FLATTENED, not dropped — the diagram is usually still there

This is the single most misleading thing about the conversion, and it has already cost a run.

Confluence renders diagrams and code blocks as *macros*. `convert_to_markdown` emits each
macro's **parameters as bare text glued to the front of its first content line**, and drops
the fence. A real example from page 2330297141:

```
graphqlHLD\_Codetrueflowchart TB
GW["Messaging gateway<br/>e.g. Infobip / WhatsApp"] -->|publishes message| SQS[(...)]
```

That is a complete mermaid flowchart. `flowchart TB` is the real first line; `graphqlHLD\_Codetrue`
is the macro's language, title and a boolean, concatenated.

So:

- **Never search for a ```` ```mermaid ```` fence.** There is not one, and the word "mermaid"
  may not appear anywhere on the page — that macro declares its language as `graphql`. On
  2026-08-20 an agent searched for the fence, found nothing, declared both architecture
  diagrams unreadable, and marked the model PARTIAL while the source sat one token away.
- **Search for the mermaid keywords**: `flowchart`, `sequenceDiagram`, `graph TD|TB|LR|RL`,
  `classDiagram`, `erDiagram`, `stateDiagram`, `gantt`, `journey`. Fenced or not.
- **Strip the macro prefix.** The diagram starts at that keyword; everything before it on the
  line is macro parameters. Take from the keyword to the blank line.
- **A diagram whose source you recovered is READ.** Do not put it in `unreadable` and do not
  mark the model PARTIAL because a `.png` or `.drawio` attachment of the same diagram cannot
  be downloaded. The attachment is a *rendering* of what you already have.
- Only a diagram with **no textual source anywhere** is unreadable. Record it, raise a gap, and
  never reconstruct it from a caption — a diagram you inferred is a diagram nobody drew.

### The conversion also corrupts values — check before transcribing

- **Markdown escapes leak into identifiers.** `\*.paceai.heliosnissan.net`, `HLD\_Code`,
  `\*Engage`. Un-escape `\*`, `\_`, `\-`, `\.` before any value goes into a table, ARN,
  hostname or config key.
- **Characters are dropped around inline code in table cells.** The same page yielded
  `` ``` rn:aws:sqs:us-east-1:422774525419:… `` — the leading `a` of `arn` gone, while the DLQ
  ARN in the very next cell survived intact.
- So **sanity-check structured values**: an ARN must start with `arn:`, a URL must have a
  scheme, a CIDR must parse. One that does not was damaged in conversion — raise a gap naming
  the cell rather than passing the broken value downstream into IAM or config.

### A linked page you cannot fetch by id — try title and space key

`confluence_get_page` takes **either** a `page_id` **or** a `title` plus a `space_key`. A
solution document routinely links to another page by title alone, with no numeric id.

On 2026-08-20 the design linked to *"CDK Existing VPC, SUBNET, GROUP IDS Details - Nissan
Optimise"*. The agent saw no page id, gave up, and raised a MISSING gap — so the VPC ids,
subnet ids and security-group ids for three environments were simply absent from the run.
They were one call away: the title was in the text and the space key (`PACEP`) was in the
metadata of the page already fetched.

So: **no id → retry with `title` and the current page's `metadata.space.key`.** Only when
*that* fails is it a `MISSING` gap, and then say both things you tried.

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
