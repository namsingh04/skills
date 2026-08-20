---
name: "gap-file-publication"
description: "Assemble the design gaps raised across every stage into one reviewable file, deduplicated, ordered by what actually blocks progress, and decide whether a human needs to see it at all. Use by the gap presenter and the gap decision agents."
version: 1
created: "2026-08-20"
updated: "2026-08-20"
---

# Gap file publication

Every stage raises gaps into its own envelope. You gather them into one file a human will
open, answer, and upload back.

The reviewer's attention is the scarce resource in this workflow. A file with forty entries
gets skimmed; a file with six gets answered.

## Gather

Read the `gaps` array from every stage output in `00-inputs/`, `10-analysis/` and `20-spec/`.
Include stages that reported `PARTIAL` — that is where most real gaps are.

## Deduplicate ruthlessly

Several agents read the same documents and find the same ambiguity. The requirements analyst,
the integration architect and the infrastructure designer will all notice that the retry
policy is unspecified.

Merge them into one entry:

- keep the clearest statement of the question,
- list every stage that raised it in `raisedBy`,
- keep the union of what they need it for in `impact`,
- take the **highest** severity and the **strongest** flags — if any raiser said it blocks
  code generation, it blocks.

A duplicate question gets one answer and one deferral, and afterwards nobody can tell which
of the two the reviewer meant.

## Remove what is not a reviewer's question

Drop, and record in `filtered` with the reason:

- **Deployment-time unknowns** — account ids, bucket names, endpoints, ARNs, secret
  references. These are validated configuration inputs, not design gaps. They belong in
  `Infrastructure.json`'s `configurationInputs`.
- **Anything answerable by reading** a file in the repository or a document already fetched.
- **Design decisions the specification stage owns** — module placement, naming, test
  structure.
- **Gaps whose `proposedDefault` is obviously right and reversible.** Record the default,
  note it in the summary, let the reviewer object on the pull request.

Keep the ones where a wrong guess produces code that has to be thrown away.

## Order by consequence

Blocking gaps first, then HIGH severity, then the rest. A reviewer who answers only the first
three should have answered the three that mattered.

## The file

Write `30-gaps/Gaps-For-Review.json`:

```json
{
  "runId": "",
  "generatedAt": "",
  "instructions": "Fill in `resolution` for each gap you want to answer. Leave it blank or write DEFER to carry a gap forward unresolved. Save and upload this file to the review form.",
  "summary": {"total": 6, "blocking": 1, "high": 2},
  "gaps": [
    {
      "id": "GAP-spec-004",
      "type": "CONTRADICTION",
      "severity": "HIGH",
      "blocksCodeGeneration": true,
      "raisedBy": ["IntegrationDesignAgent", "FunctionalRequirementAgent"],
      "question": "Solution §3 specifies at-least-once delivery; §7 specifies exactly-once. Which applies to the ingest path?",
      "context": "§3.2 'Ingest flow' vs §7.1 'Delivery guarantees'. The two imply different dedupe requirements.",
      "impact": "determines whether an idempotency key is required on every handler",
      "proposedDefault": "at-least-once with idempotency keys, per §3 which is the more detailed section",
      "resolution": ""
    }
  ],
  "filtered": [{"question": "what is the target account id?", "why": "deployment-time configuration, not a design decision"}]
}
```

`resolution` is always empty. You never answer your own gaps — including the ones where you
are confident, because a confident answer recorded as a human decision is a fabrication.

## The decision: does a human need to see this?

A separate, minimal judgement, answered from the file you just wrote.

**Answer from the `gaps` array itself**, entry by entry. Any entry with
`requiresHumanInput: true` or `blocksCodeGeneration: true` means yes. An empty array is a
legitimate no.

**Never trust a summary count over the array.** If a count says two and the array holds three
flagged entries, the array wins. Both stages have shipped counts that disagreed with their own
arrays, which is why the array is the rule and the count is not.

**When in doubt, say yes.** A review form a reviewer dismisses in ten seconds is cheap.
Skipping the review sends unreviewed ambiguity into code generation — that has happened on
this workflow and was not noticed until the generated code was wrong.

## Applying resolutions

When the reviewer's file comes back, write `30-gaps/Gap-Resolutions.json` with each gap's
`resolution` as they wrote it.

- **Apply exactly what it says. Nothing more.** An answer about one field is about one field.
  Extending it to "everything similar" is inventing three decisions from one.
- A blank or `DEFER` resolution stays unresolved and keeps its `proposedDefault`. Record it as
  deferred, not resolved — the summary distinguishes them and so should you.
- An answer that contradicts a higher authority is still the answer: the reviewer knows more
  than the document. Record the override.
- An answer you cannot interpret is not a licence to guess. Keep the gap unresolved, record
  the reviewer's words verbatim, and note that it could not be applied.
