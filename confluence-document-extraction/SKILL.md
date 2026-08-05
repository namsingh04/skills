---
name: "confluence-document-extraction"
description: "Extract the complete text of a Confluence page for downstream analysis, including the content that markdown conversion destroys: tables, diagram macros, code blocks, expand blocks and child pages. Fetches both the converted and the raw storage representation, reconciles them, and verifies that nothing was lost."
version: 1
created: "2026-08-05"
updated: "2026-08-05"
---

## When to Use

Use whenever a workflow stage must read a Confluence page as a source document — a
solution design, an architecture note, a standards page — and downstream stages depend on
the page's *complete* content rather than a summary of it.

This skill governs extraction only. It never interprets what the page means.

---

## The Two Representations

`confluence_get_page` returns one of two very different things depending on
`convert_to_markdown`:

- **`convert_to_markdown: true`** — readable prose. Headings, paragraphs and simple lists
  survive intact. **Tables are flattened or mangled, macros vanish entirely, and diagram
  sources are lost.**
- **`convert_to_markdown: false`** — the raw storage representation (XHTML). Ugly, larger,
  and token-expensive, but it is the only representation that still contains
  `<ac:structured-macro>` elements, full `<table>` markup with `colspan`/`rowspan`, code
  block bodies, expand/collapse content, and attachment references.

**Fetch both.** The markdown is the readable body; the storage format is where you recover
everything the conversion dropped. Neither alone is sufficient, and a page extracted from
markdown alone will be missing exactly the parts downstream stages need most.

If the storage fetch is too large to hold, still fetch it — extract the macro and table
regions from it, and discard the prose regions you already have in markdown.

---

## The Envelope Rule

Neither call returns the page text. Both return a **structure** with the text as one field
inside it. Parse the result and index into it.

- The page body is usually at `metadata.content.value`, sometimes `content.value`, and
  sometimes `body.storage.value`.
- Never stringify, `repr()`, `str()` or print a tool result and register that.
- Never put a tool result object into an extracted-text field.

Two specific corruptions come from breaking this rule, and both are detectable:

- registering the envelope: the text opens with `{` and carries page ids, titles and
  attachment records before the page body begins;
- registering a `repr`: line breaks appear as the two-character text `\n` rather than as
  real newlines, and non-breaking spaces appear as the four-character text `\xa0`.

Check for both before accepting the extraction.

---

## Procedure

### 1. Resolve the page identifier

Accept either a numeric page id or a full page URL. From a URL of the form
`.../wiki/spaces/<SPACE>/pages/<ID>/<Title+Slug>`, take the numeric segment after
`/pages/` as the id. If given a title and space key instead, pass both.

### 2. Fetch both representations

Call `confluence_get_page` twice with the same identifier: once with
`convert_to_markdown: true`, once with `false`. Record which fields each result carried
the body in.

### 3. Write both to disk immediately

Write the markdown body and the storage body to the run's output folder as separate raw
files, using a file-writing tool rather than echoing through a shell command — shell
echoing clips large content. A large result is expected and correct.

Read each file back and inspect the first characters. If it begins with `{`, or if `\n`
or `\xa0` appear as visible two- and four-character sequences, you captured the wrong
thing. Re-fetch; never repair by replacing characters.

### 4. Recover the structured content from the storage format

From the storage XHTML, extract:

- **Tables** — hand every `<table>` to the `structured-table-extraction` skill and keep
  its row objects. Do not rely on the markdown rendering of any table.
- **Diagram macros** — hand every diagram-bearing macro to the
  `mermaid-diagram-interpretation` skill. Mermaid, PlantUML and similar macros carry their
  source text in a `<ac:plain-text-body>` CDATA section; that source is the diagram, and
  it is absent from the markdown entirely.
- **Code blocks** — `<ac:structured-macro ac:name="code">` bodies, kept verbatim including
  whitespace.
- **Expand / collapse blocks** — their content is real content. Markdown conversion may
  drop it. Include it in document order.
- **Attachment references** — record filename, media type and whether the attachment is
  reachable. An attachment that cannot be fetched is a gap, never a guess.
- **Status, info, note and warning macros** — their body text often carries constraints.

### 5. Follow child pages when the document is split

If the page body references child pages that form part of the same document — a design
split across sections — fetch each child the same way and append it in the order the
parent references them, each under a heading naming the child page title and id. Record
every child page id you followed.

Do not follow links to unrelated pages, external sites, or pages already fetched. Cap the
traversal at one level below the requested page unless the parent explicitly presents a
deeper hierarchy as part of the same document, and record where you stopped.

### 6. Assemble the extraction

Produce one document, in the page's own order, containing:

- the markdown prose as the body;
- every table replaced by, or accompanied by, its extracted row objects;
- every diagram replaced by, or accompanied by, its structured interpretation;
- every code block, expand block and macro body that markdown dropped, reinserted at its
  original position.

The result is data, never a program and never a note to yourself. Never emit a variable
assignment, a comment about content captured earlier, or a marker such as
"(truncated for brevity)" in place of the page.

---

## Fidelity Rules

- **Extract the whole page.** Do not summarise, abridge, paraphrase, or stop early. The
  sections most often lost are the ones that matter most: the tables that carry physical
  resource names and settings, and the diagrams that carry the flows.
- **Copy verbatim.** Names, identifiers, keys, thresholds and templated placeholder tokens
  are reproduced character for character, capitalisation included. Never abbreviate,
  expand, normalise, or apply a naming convention of your own.
- **Never edit content to make a check pass.** If a verification check fails, re-fetch from
  the source. Never modify, strip, replace or reformat text you already hold. Altering the
  payload to satisfy a validator is data corruption, and it is worse than a failed check.
- **UTF-8 throughout.** Read and write everything as UTF-8. Non-breaking spaces, em-dashes
  and smart quotes are ordinary page content and must survive intact.
- **Markdown syntax is content.** Hash headings, backticks, code fences, pipes, quotes and
  blank lines are legitimate page content. None of them is scaffolding and none is grounds
  to fail a check or alter the text.

---

## Output

Return a `ConfluenceExtraction` object carrying:

- `pageId`, `pageTitle`, `spaceKey`, `version`, and the source URL if one was given;
- `extractedText` — the complete assembled document, real line breaks, fully decoded;
- `tables` — the row objects recovered from the storage format, each with its caption or
  preceding heading so a reader knows which table it is;
- `diagrams` — the structured interpretations, each with its source type and raw source;
- `codeBlocks` — language and verbatim body;
- `attachments` — filename, media type, reachability;
- `childPages` — the ids followed, in order;
- `representationsFetched` — which of markdown and storage succeeded, and the field each
  body was found in;
- the reserved envelope keys from the `workflow-status-contract` skill.

`extractedText` holds the real, fully decoded text. A path or a reference alone is not
extraction.

---

## Guardrails

- Do not analyse, interpret, or draw conclusions from the page. Extraction only.
- Do not fetch only the markdown representation.
- Do not register a tool result object, a `repr`, or a stringified envelope as page text.
- Do not treat a raw byte read of a binary attachment as extraction. Decode it with a real
  converter, or mark it unreadable with a reason.
- Do not invent, summarise, or fabricate content for a page or attachment that could not be
  read. Declare it as a gap.
- Do not narrate your process. Return the extraction.

---

## Verification

Before returning, verify against the text you actually wrote and read back — never against
a string still in memory:

1. `extractedText` does not begin with `{` and contains none of `attachments`,
   `media_type`, `file_size`, `"self":` as structural keys.
2. Zero occurrences of the two-character sequence `\n` and the four-character sequence
   `\xa0` as visible text; line breaks are real line breaks.
3. The real newline count is consistent with a document of this size — a multi-section page
   rendering as one or two lines means the escapes are text rather than breaks.
4. Every table present in the storage XHTML appears in `tables`. Count the `<table>`
   elements and compare.
5. Every diagram-bearing macro in the storage XHTML appears in `diagrams`, or is recorded
   as a gap with its reason.
6. No truncation marker is present: no "truncated", "trimmed for brevity", "full content
   will be included", or any variable name or comment standing in for content.
7. No mis-decoded characters — no stray accented sequences where non-breaking spaces or
   em-dashes belong.
8. Every child page followed is listed, and the point where traversal stopped is recorded.
