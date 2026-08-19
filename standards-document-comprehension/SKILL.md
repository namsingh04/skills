---
name: standards-document-comprehension
description: Turn a coding standards document - patterns, code architecture, naming conventions, required checks - into rules a generator can actually obey and a validator can actually check. Use when reading the standards file or page.
---

# Standards document comprehension

A standards document is written for humans, who read it once and absorb it. A code generator
needs something narrower: statements it can check itself against, one at a time.

Your output is a rulebook, not a summary.

## The four kinds of rule

**Structural** — where files go, how modules are organised, what a package contains.
Checkable against a path. *"Handlers live in `src/handlers/`, one file per operation."*

**Naming** — casing, prefixes, suffixes, plurality. Checkable against an identifier.
*"Test files are named `test_<module>.py`."*

**Pattern** — how a recurring problem is solved here: error handling, configuration,
logging, dependency injection, how external calls are wrapped. Checkable by comparison with
an example. This is the category most standards documents state loosely and most code
generators ignore.

**Process** — what must be run and pass before code is considered done: lint, type check,
coverage floor, required review. Checkable by running it.

## Method

**1. Extract statements, not sections.** Each rule is one obligation. A paragraph usually
contains three. Split them.

**2. Make each rule checkable, or mark it advisory.** *"Code should be maintainable"* cannot
be checked and will not be followed. If the document gives a concrete form of it elsewhere,
use that; if it does not, record it as `advisory` and move on. Do not invent a concrete
version — an invented rule is indistinguishable from a real one once it is in the rulebook,
and it will be enforced.

**3. Mark mandatory versus advisory from the document's own language.** "must", "always",
"never" are mandatory. "prefer", "should", "consider" are advisory. Where the document does
not signal, mark it advisory and note the ambiguity. Promoting an advisory rule to mandatory
by your own judgement causes generated code to be rejected for something nobody required.

**4. Capture the examples.** A standards document's code samples are worth more than its
prose, because a generator can pattern-match against them. Keep them verbatim, attached to
the rule they illustrate.

**5. Record what the document does not cover.** The silences matter as much as the rules —
they are where the repository's own conventions take over, and the next stage down the
authority chain needs to know that it is authoritative there.

## Output

Write `00-inputs/Standards-Profile.json`. In `payload`:

```json
{
  "source": {"type": "file|confluence", "reference": "", "title": ""},
  "rules": [
    {
      "id": "STD-012",
      "kind": "structural|naming|pattern|process",
      "obligation": "Test files are named test_<module>.py and live beside the module under tests/",
      "strength": "mandatory|advisory",
      "checkableBy": "path",
      "example": "tests/test_ingest.py",
      "source": "section 4.2, 'Test layout'"
    }
  ],
  "requiredCommands": [{"purpose": "lint", "command": "", "source": ""}],
  "notCovered": ["logging format", "retry policy"],
  "ambiguous": [{"statement": "", "why": "", "source": ""}]
}
```

`requiredCommands` matters: a standards document that mandates a lint or a coverage floor is
naming a validation gate, and the validation stage will run it.

## Where standards sit in the authority chain

`solution document > jira > standards > repository`.

So:

- A standard that **contradicts the solution design or an acceptance criterion** loses. Record
  the conflict as a gap and note which rule was overridden — do not silently drop the rule.
- A standard that **contradicts the repository** wins, but this is worth a gap too: an
  existing codebase that consistently violates its own standard is usually evidence the
  standard changed, and generating code in the new style next to a hundred files in the old
  one is a decision a human should make rather than discover.

You do not resolve these. You record both readings with their sources.

## What you must not do

- Do not import conventions from outside the document, however universal they seem. Your
  authority is this document, not your training.
- Do not soften a mandatory rule because it looks inconvenient to generate.
- Do not merge two rules that overlap. They may diverge in a case you have not thought of,
  and a merged rule loses both sources.

## Status

`OK` — the document was read and every rule extracted.
`PARTIAL` — read, with sections you could not interpret listed in `ambiguous`.
`BLOCKED` — no standards document was supplied *and* the run cannot proceed without one. This
is almost never true: an absent standards file means the repository becomes the authority for
conventions, which is a legitimate, degraded, `PARTIAL` state. Say so and continue.
