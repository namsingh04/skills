---
name: "structured-table-extraction"
description: "Convert tables from markdown or XHTML source into lossless row objects, preserving merged cells, multi-line cells, embedded code and lists, and templated placeholder tokens. Produces structured rows that downstream stages can read field by field instead of re-parsing prose."
version: 1
created: "2026-08-05"
updated: "2026-08-05"
---

## When to Use

Use whenever a source document carries information in a table that a downstream stage must
read field by field — a resource inventory, a configuration matrix, a mapping of names to
settings, a traceability grid.

Tables are where documents put the values that matter: physical resource names, stack
assignments, retention periods, timeouts, key names, event pattern literals. A table read
as prose loses the column each value belonged to, and a value without its column is
useless.

---

## Why Not Just Read the Markdown

Markdown rendering of a table is lossy in ways that are easy to miss:

- a cell containing a newline, a bulleted list, or a code block is collapsed onto one line
  or split across rows;
- merged cells (`colspan` / `rowspan`) have no markdown representation at all, so values
  silently shift into the wrong columns;
- a cell containing a pipe character breaks the row;
- long tables may be truncated by the converter;
- formatting-only cells vanish, changing the column count.

When the source has an XHTML (storage-format) representation, parse **that**, and use the
markdown only as a cross-check on row and column counts.

---

## Procedure

### 1. Locate the table and its identity

Record what identifies this table: its caption, or the nearest preceding heading, plus its
ordinal position in the document. Downstream stages need to know *which* table a row came
from, and headings differ between documents — never assume a particular title.

### 2. Establish the header row

Take the header cells verbatim, in order, as the field names. Trim surrounding whitespace
only. Do not rename, normalise case, expand abbreviations, or map them onto names you
expect. A column called `Stack Name` stays `Stack Name`.

If the table has no header row, use positional field names `col1`, `col2`, … and record
`headerRowPresent: false`.

If the table has multiple header rows, join them per column with a single space, in
document order, and record `headerRows: <n>`.

### 3. Expand merged cells before mapping

Before mapping any row to fields, expand spans so every logical row has one value per
column:

- a cell with `colspan="n"` fills the next `n` columns with the same value;
- a cell with `rowspan="n"` fills the same column in the next `n` rows with the same value.

Do this first. Mapping rows to fields before expanding spans is what puts values in the
wrong columns, and the result looks plausible enough to survive review.

### 4. Extract each cell losslessly

For every cell:

- keep the text **verbatim**, including capitalisation, punctuation, and templated
  placeholder tokens such as an environment placeholder in double curly braces —
  reproduce those byte for byte, never resolved, never substituted, never allowed to
  become the string `null`;
- preserve internal line breaks as real newlines;
- a cell containing a list becomes an array of item strings;
- a cell containing a code block keeps its body verbatim, whitespace included;
- an empty cell becomes an empty string, never a guess, never `null` where a name belongs.

### 5. Emit rows

One object per logical row, keyed by the header field names, in document order. Row order
is meaningful; preserve it.

---

## Fidelity Rules

- **Every row, every column.** Include rows whose content appears irrelevant to the current
  task. Filtering is a downstream decision; your job is to lose nothing. A value dropped
  here cannot be recovered by any later stage.
- **Never normalise a name.** Do not abbreviate, expand, re-case, or apply a naming
  convention. If the table says one thing and convention suggests another, the table wins
  and the convention is irrelevant.
- **Never merge or split rows** to make the shape tidier.
- **A pipe, quote, brace or backtick inside a cell is content.** Escape it properly when
  serialising; never let it break the structure and never strip it.

---

## Output

Return a `TableExtraction` object:

```json
{
  "tables": [
    {
      "tableId": "<document-order index>",
      "caption": "<caption or nearest preceding heading, verbatim>",
      "sourceRepresentation": "xhtml | markdown",
      "headerRowPresent": true,
      "headerRows": 1,
      "fields": ["<header cell>", "..."],
      "rowCount": 0,
      "rows": [
        { "<header cell>": "<verbatim value>" }
      ],
      "spansExpanded": true,
      "notes": ["<anything irregular about this table>"]
    }
  ]
}
```

Plus the reserved envelope keys from the `workflow-status-contract` skill.

Every value in a `TableExtraction` must be a properly escaped JSON string. Never splice
pipe-delimited rows or nested backticks into a JSON string unescaped.

---

## Guardrails

- Do not interpret the table's meaning, classify its rows, or decide what is in scope.
  Extraction only.
- Do not summarise a large table or emit a representative subset.
- Do not drop a column because it looked like formatting.
- Do not fabricate a header name the table does not have.
- Do not resolve a templated placeholder token.
- Do not emit a literal `null` where a name or identifier belongs.

---

## Verification

Before returning, verify:

1. `rowCount` equals the number of data rows in the source, excluding header rows.
2. Every row object has exactly one entry per field name — no row is short, none is long.
3. If the source had `colspan` or `rowspan` attributes, `spansExpanded` is true and the
   affected columns carry the repeated value.
4. Every templated placeholder token appears exactly as the source writes it, and no value
   is the string `null` or begins with `null-`.
5. Multi-line cells contain real newlines, not the two-character text `\n`.
6. The extraction parses as valid JSON with balanced braces and brackets.
7. Where both representations were available, the row and column counts from the XHTML
   match the markdown; any mismatch is recorded in `notes` and the XHTML is used.
