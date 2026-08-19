---
name: additional-instruction-triage
description: Decide what a free-text "additional information" instruction means for this run - mandatory in scope, a narrowing of scope, an out-of-scope exclusion, or nothing at all. Use when the run supplies extra operator instructions alongside the formal inputs.
---

# Additional instruction triage

The operator can type an instruction into a free-text field. It carries real authority — a
human wrote it about *this run*, knowing what the documents say — but it is unstructured, and
misreading it is expensive in both directions: ignoring a mandate produces the wrong thing;
inventing a mandate from a passing remark produces things nobody asked for.

## First: is there an instruction at all?

The field is optional, and an empty one is the common case. Treat all of these as **no
instruction**, and do not mention them again:

`NA`, `na`, `N/A`, empty, whitespace, `null`, `undefined`, `none`, or unresolved template
text containing `{{` or `}}`.

Anything else is an instruction and must be triaged.

## The four categories

**MANDATE** — something to do that the documents do not already require.
*"Add a dead-letter queue to the ingest path."*
→ Mandatorily in scope. It joins the requirements as though the solution document had said
it, and it is traced to the instruction rather than to a document.

**NARROWING** — a restriction to part of what the documents describe.
*"Only the ingestion Lambda this time."* / *"Just the API layer."*
→ In scope **and** it removes everything else. This is the category most often missed: an
instruction that names a subset is not merely emphasis, it is an exclusion of the rest. Record
what is now out of scope explicitly, because a later stage reading the full solution document
will otherwise build all of it.

**EXCLUSION** — something explicitly not to do.
*"Don't touch the auth module."* / *"No infrastructure changes."*
→ A hard boundary. Any requirement that would cross it becomes a gap, not a quiet override.

**CONTEXT** — background, a preference, or a remark with no obligation in it.
*"This is for the Q3 migration."* / *"The team prefers small commits."*
→ Record it, act on nothing. A preference is not a requirement, and treating one as a mandate
is how runs acquire work nobody asked for.

## When it is genuinely unclear

Ambiguity here is common and cheap to surface. Raise a gap of type `AMBIGUITY` with both
readings and a `proposedDefault` of the **narrower** interpretation.

Narrower is the right default: doing less than asked is visible and correctable in an amend
run; doing more than asked means unreviewed code in a repository, and nobody notices until
it ships.

## Conflict with the documents

The instruction is written **after** the documents and **about this run**, so where they
conflict on scope, the instruction generally wins — but never silently.

- Instruction narrows the documents → follow the instruction, record what was excluded.
- Instruction contradicts an acceptance criterion → **raise a gap**. Acceptance criteria
  define what "done" means; an instruction that makes them unmeetable is a decision a human
  must confirm, not one you take.
- Instruction contradicts a mandatory standard → raise a gap, note the standard it overrides.

## Output

Write `00-inputs/Additional-Instruction.json`. In `payload`:

```json
{
  "present": true,
  "raw": "the instruction exactly as typed, unedited",
  "classification": "MANDATE|NARROWING|EXCLUSION|CONTEXT|NONE",
  "derivedRequirements": [{"id": "AI-001", "statement": "", "mandatory": true}],
  "inScope": ["ingestion lambda"],
  "outOfScope": ["reporting api", "notification handler"],
  "boundaries": ["do not modify src/auth/**"],
  "conflicts": [{"with": "jira:ABC-124 AC-2", "nature": "", "raisedAsGap": "GAP-inputs-3"}]
}
```

Keep `raw` verbatim. Every later stage that acts on this should be able to see the words the
operator actually typed, not your reading of them — and when the run does something
surprising, that field is the first thing anyone will look at.

## What you must not do

- Do not extend an instruction beyond its words. "Add retries to the ingest call" is about
  the ingest call, not about every external call in the system.
- Do not treat a question as an instruction. *"Should we use a queue here?"* is a gap.
- Do not let a `CONTEXT` remark quietly shape the design. If it matters enough to act on, it
  is a `MANDATE` and you should classify it as one; if you are not confident, raise a gap.
