---
name: "mermaid-diagram-interpretation"
description: "Read Mermaid diagrams embedded in documents and turn them into structured nodes, edges, participants and flows that downstream analysis can consume. Covers extraction of the diagram source from macro markup, parsing of the common diagram grammars, and honest declaration of diagrams that are images rather than source."
version: 1
created: "2026-08-05"
updated: "2026-08-05"
---

## When to Use

Use whenever a source document contains a diagram that carries architectural meaning —
component relationships, event flows, sequence of calls, deployment topology — and a
downstream stage needs those relationships as data rather than as a picture.

Diagrams routinely state things the prose does not: which component calls which, in what
direction, what the failure path is, which queue feeds which consumer. A document read
without its diagrams is missing its flows.

---

## Finding the Diagram Source

A diagram is only readable if its **source text** is present. Where to look:

- **Macro markup.** In a storage-format (XHTML) document, a diagram macro appears as an
  `<ac:structured-macro>` element whose body holds the source in a
  `<ac:plain-text-body>` CDATA section. That CDATA content is the diagram.
- **Fenced code blocks.** In markdown, a diagram may appear as a fenced block tagged
  `mermaid` (or the tool's own tag). The fence body is the diagram.
- **Attached source files.** A diagram may be an attachment with a text source
  (`.mmd`, `.puml`, or an XML-based editor format).

**Markdown conversion destroys diagram macros entirely.** If the document was converted to
markdown, the diagram is simply gone from it — you must read the storage representation to
find the source at all. Never conclude a document has no diagrams from its markdown alone.

### When there is no source

A diagram that exists only as a rendered image — PNG, JPEG, or an exported SVG with no
recoverable structure — cannot be read as data.

**Declare it as a gap.** Record the attachment filename, its position in the document, and
any caption or surrounding text. Never guess at what an image depicts, and never
reconstruct a flow from a caption. A guessed architecture that looks plausible is more
damaging than a declared gap, because nothing downstream will question it.

---

## Parsing

Take the first token of the source to identify the grammar, then extract accordingly.

### Flowchart / graph

Header `flowchart <direction>` or `graph <direction>`.

- **Nodes.** `id[Label]`, `id(Label)`, `id{Label}`, `id[[Label]]`, `id[(Label)]` and
  similar. Record the node id, the label verbatim, and the shape — the shape is often
  meaningful (a cylinder for storage, a diamond for a decision).
- **Edges.** `A --> B`, `A -.-> B`, `A ==> B`, `A --- B`, and their labelled forms
  `A -->|label| B` or `A -- label --> B`. Record source, target, the label verbatim, the
  line style, and whether it is directed.
- **Direction is meaning.** `A --> B` and `B --> A` describe opposite flows. Preserve it.
- **Subgraphs.** `subgraph <name> … end` groups nodes; record the grouping, since it
  usually maps to a stack, a service boundary, or an account.

### Sequence diagram

Header `sequenceDiagram`.

- **Participants.** `participant X as Label`, `actor X`. Record id and label.
- **Messages.** `A->>B: text`, `A-->>B: text` (dashed, typically a response),
  `A-)B: text` (async). Record sender, receiver, the message text verbatim, whether it is
  a response, and its **ordinal position** — order is the entire point of a sequence
  diagram.
- **Blocks.** `alt` / `else` / `opt` / `loop` / `par` / `critical` with their condition
  text. These are the conditional and retry behaviour; capture the condition and which
  messages fall inside each block.
- **Notes.** `Note over A,B: text` frequently carries a constraint.

### State diagram

Header `stateDiagram` or `stateDiagram-v2`. Record states, transitions with their trigger
labels, and the initial and terminal states (`[*]`).

### Entity relationship diagram

Header `erDiagram`. Record entities, their attributes, and relationships with their
cardinality notation preserved verbatim.

### C4 diagrams

Headers beginning `C4Context`, `C4Container`, `C4Component`. Record each element's alias,
label, technology and description, and every `Rel(...)` as a directed edge with its label
and technology.

### Class diagram

Header `classDiagram`. Record classes, members, and relationships with their arrow types
preserved, since the arrow type is the relationship semantics.

---

## Other Diagram Grammars

If the source is PlantUML, Graphviz DOT, or an XML-based editor format rather than
Mermaid, apply the same principle: extract participants/nodes, directed edges with their
labels, and grouping. Record `sourceType` accurately. Do not attempt to convert one
grammar into another — extract the relationships and move on.

For an XML editor format whose payload is compressed or encoded rather than plain text,
treat it as an image-only diagram: declare the gap.

---

## Mapping to the Domain

After parsing, and only after, map what the diagram states onto the domain the document is
about — for infrastructure work, onto services, resources and flows.

- Map only what the diagram actually says. A node labelled with a service name identifies
  that service; a node labelled with a role or a component name does not imply a
  particular implementation.
- **Never infer a resource the diagram does not show.** A diagram showing a queue does not
  imply a dead-letter queue, an alarm, or a retry policy unless it shows them.
- Where a node's label matches a name stated elsewhere in the document, link them, and say
  which name you matched. Where it does not match anything, keep the label verbatim and
  say so.
- Contradictions between the diagram and the prose are **warnings**, not silent
  resolutions. Record both readings with their sources.

---

## Output

Return a `DiagramInterpretation` object:

```json
{
  "diagrams": [
    {
      "diagramId": "<document-order index>",
      "caption": "<caption or nearest preceding heading>",
      "sourceType": "mermaid | plantuml | dot | image-only | unknown",
      "grammar": "flowchart | sequenceDiagram | stateDiagram | erDiagram | c4 | classDiagram | null",
      "rawSource": "<the diagram source verbatim, or an empty string when image-only>",
      "nodes": [ { "id": "", "label": "", "shape": "", "group": "" } ],
      "edges": [ { "from": "", "to": "", "label": "", "style": "", "directed": true, "ordinal": 0 } ],
      "blocks": [ { "type": "alt|opt|loop|par", "condition": "", "containsEdges": [] } ],
      "domainMapping": [ { "node": "", "mappedTo": "", "basis": "" } ],
      "unmapped": ["<labels that matched nothing stated elsewhere>"],
      "notes": []
    }
  ]
}
```

Plus the reserved envelope keys from the `workflow-status-contract` skill. Image-only
diagrams appear here with `sourceType: "image-only"` **and** as a gap in `gaps`.

---

## Guardrails

- Do not describe a diagram in prose in place of extracting its structure.
- Do not invent nodes, edges, or flows that the source does not contain.
- Do not reverse or drop edge direction, and do not lose sequence ordering.
- Do not resolve a templated placeholder appearing in a label; reproduce it byte for byte.
- Do not silently skip a diagram you could not parse — record it with its reason.
- Do not guess the contents of an image-only diagram from its filename or caption.

---

## Verification

Before returning, verify:

1. Every diagram-bearing macro or fenced diagram block found in the source appears in
   `diagrams`, or is recorded as a gap.
2. `rawSource` is present and verbatim for every diagram that had a text source.
3. Node and edge counts match the source: count the edge operators in `rawSource` and
   compare with `edges`.
4. Every edge names nodes that exist in `nodes`.
5. Sequence-diagram messages carry ordinals that reflect source order.
6. Every conditional block's condition text is captured verbatim.
7. Every `domainMapping` entry states the `basis` for the mapping; nothing is mapped on a
   hunch.
8. Every image-only diagram appears in `gaps` with `requiresHumanInput: true`.
9. The output parses as valid JSON with balanced braces and brackets.
