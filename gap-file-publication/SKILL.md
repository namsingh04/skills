---
name: "gap-file-publication"
description: "Assemble the design gaps raised across every stage into one reviewable file, deduplicated, ordered by what actually blocks progress, and decide whether a human needs to see it at all. Use by the gap presenter and the gap decision agents."
version: 3
created: "2026-08-20"
updated: "2026-09-01"
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

- **Deployment-time unknowns that are simply not known yet** — account ids, bucket names,
  endpoints, ARNs, secret references that WILL be supplied by configuration at deploy time and
  have a clear home. These are validated configuration inputs, not design gaps. They belong in
  `Infrastructure.json`'s `configurationInputs`. **But do not drop a value that is genuinely
  ABSENT or UNCONFIRMED for an environment the run must target and whose absence blocks
  deployment or correctness** — e.g. network IDs missing for some of the required environments,
  a permission the role does not yet have, an egress path nobody has confirmed. That is a
  reviewer's question: keep it (see the flag rule below), do not file it away as routine config.
- **Anything answerable by reading** a file in the repository or a document already fetched. This
  catches the class that keeps slipping through: a **request/response schema, payload shape, a header
  or Content-Type, an endpoint URL, a request format, an enum, a parameter's meaning** is almost always
  in the **solution document's mock data and sample exchanges** — search `Solution-Model.json` for the
  noun before publishing it. A run published five such gaps whose answers the solution doc stated
  dozens of times over. If the value is in the model, drop the gap to `filtered` with "present in
  Solution-Model" as the reason; do not send the reviewer a question the document already answers.
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

**Set the review flags yourself — do not merely inherit them.** A gap that survived filtering is,
by definition, one where a wrong guess produces code that has to be thrown away, so it needs an
answer. On every gap you publish:

- Set **`requiresHumanInput: true`** when the gap is `HIGH` severity, OR is a `MISSING` /
  `UNCONFIRMED` value the reviewer must supply before the code is correct or deployable (the
  absent network IDs, the unconfirmed role permission, the egress question above). These are
  exactly the gaps that must open the review gate.
- Set **`blocksCodeGeneration: true`** additionally when the code literally cannot be written
  correctly without the answer (a contradiction in the contract, an undecided delivery guarantee).
  A value that only blocks *deployment* needs `requiresHumanInput`, not necessarily this.
- The raisers often leave both flags unset; that is not permission to publish an unflagged HIGH
  gap. If you kept it, flag it.

**`summary` must agree with the array.** Count `blocking` as the number of entries with
`blocksCodeGeneration: true`, and make sure at least the `requiresHumanInput` entries are visible in
the summary too. A file with HIGH gaps and `blocking: 0` and no `requiresHumanInput` is the exact
shape that silently skips the gate — never emit it.

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
