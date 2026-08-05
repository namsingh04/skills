---
name: "jira-issue-extraction"
description: "Extract the full description of one or many Jira issues and assemble them into a single ordered document for downstream analysis. Handles comma-separated key lists, Atlassian Document Format descriptions, attachments and comments, and verifies that what was registered is the ticket text rather than the API envelope."
version: 1
created: "2026-08-05"
updated: "2026-08-05"
---

## When to Use

Use whenever a workflow stage takes Jira issues as a source document — acceptance
criteria, enabler descriptions, requirement tickets — and downstream stages read the
assembled text rather than calling Jira themselves.

One issue or fifty: the procedure is the same.

---

## Inputs

A list of issue keys, given as a comma-separated string. Split on commas, trim each key,
drop empty entries, and **process them in the order given**. List order is document order
and it is meaningful.

If the list is blank, that is a recorded gap, not a failure: register an empty extraction,
name the missing input in `warnings` and in `gaps`, and return `status: "PARTIAL"`. Never
substitute a previous run's issues, never derive criteria from another document, and never
read a key off disk.

---

## The Envelope Rule

`jira_get_issue` does not return the description. It returns a structure with the
description as one field inside it. Parse the result and index into it.

- The description is at `fields.description`, and on some simplified responses at
  `description`.
- The summary is at `fields.summary`, or `summary`.
- Never stringify, `repr()`, `str()` or print a tool result and register that.

A registration that opens with `{` and carries `issuetype`, `customfield_…`,
`renderedFields`, `"self":`, `avatarUrls` or `statusCategory` is the envelope, not the
ticket.

---

## Reading a Description

`fields.description` comes back in one of two shapes.

**A string.** That is the text. Take it verbatim.

**An object.** That is Atlassian Document Format. Walk the node tree and concatenate the
text values, emitting a real newline at each paragraph, heading, list item, table row and
`hardBreak`, so the result reads as plain text with real line breaks.

- Preserve list markers and heading text; the structure carries meaning.
- Table nodes inside ADF are content — walk them rather than skipping them, and keep cell
  order.
- Code blocks (`codeBlock`) keep their body verbatim.
- Never JSON-dump the ADF object into the extracted text.
- If plain text genuinely cannot be recovered, record a gap naming that key. Do not
  paraphrase the ticket.

---

## Assembling the Document

For each key, in the order given, request the fields you need — at minimum summary and
description — and build one section.

**Each section begins with the issue key, then the description.** The first line is the
issue key followed by the issue summary; everything after it is the description text,
verbatim. If the description already opens with a line naming the key, leave it exactly as
it is rather than duplicating it.

The summary comes from the ticket's own `summary` field, so writing that line is
reconstruction from real data, not invention. Never invent a title.

Everything else is verbatim:

- no summarising, no reformatting;
- no renumbering, merging or splitting of individual criteria;
- keep every trailer line the ticket carries — dependencies, testing scope, analytics,
  accessibility, design links, test data notes. Those lines are requirements too.

Join the sections in order with a blank line between them. Write the assembled document to
the run's output folder as a raw text file, with real line breaks, using a file-writing
tool rather than shell echoing.

---

## Attachments and Comments

- **Attachments.** Record filename, media type and size for every attachment. Download and
  decode one only when the ticket's text refers to it as carrying requirement content. A
  binary attachment is decoded with a real converter, never read as raw bytes; if it cannot
  be decoded, record it as a gap.
- **Comments.** Do not fold comments into the description by default — a description is the
  agreed statement and a comment is a conversation. Capture them separately when the ticket
  text points at a comment for a value, and record which comment you took it from.

---

## Output

Return a `JiraExtraction` object:

```json
{
  "source": "jira",
  "issueKeys": ["<key>", "..."],
  "sourceIssues": [
    {
      "key": "<key>",
      "summary": "<verbatim summary>",
      "descriptionFormat": "string | adf",
      "characterCount": 0,
      "attachments": [ { "filename": "", "mediaType": "", "size": 0, "decoded": false } ]
    }
  ],
  "extractedText": "<the assembled document, real line breaks>",
  "rawFilePath": "<absolute path of the raw text file written>"
}
```

Plus the reserved envelope keys from the `workflow-status-contract` skill.

`extractedText` is what downstream stages read, and they read nothing else — a path or a
reference alone is not extraction.

---

## Guardrails

- Do not analyse, interpret, or judge the tickets. Extraction only.
- Do not reorder, deduplicate, or drop a key from the list you were given.
- Do not summarise a long description or truncate it.
- Do not register an envelope, a `repr`, or an ADF dump as text.
- Do not invent an issue summary, a criterion, or a title.
- Do not silently continue past a key that could not be fetched — record it as a gap and
  keep going with the rest.
- Do not narrate your process. Return the extraction.

---

## Verification

Run every check against the file you wrote and read back, never against a string still in
memory:

1. `issueKeys` holds exactly the keys you were given, in the order given — none dropped,
   none added, none reordered.
2. `extractedText` does not begin with `{` and contains none of `issuetype`,
   `customfield`, `renderedFields`, `"self":`, `avatarUrls`, `statusCategory`.
3. Zero occurrences of the two-character text `\n`; line breaks are real line breaks.
4. For every key in `issueKeys`, that issue's key **and** its summary text appear in
   `extractedText`.
5. Every section begins with its issue key.
6. `sourceIssues` has one entry per key, and the character counts sum to within a small
   margin of `len(extractedText)` — the difference being only the key/summary lines you
   prepended and the blank lines between sections. A large shortfall means a description
   was truncated; re-fetch that issue rather than patching the text.
7. Every key that could not be fetched appears in `gaps`.
8. The output parses as valid JSON with balanced braces and brackets.

On any failure, re-fetch the affected issues and rebuild. Never patch `extractedText` by
hand.
