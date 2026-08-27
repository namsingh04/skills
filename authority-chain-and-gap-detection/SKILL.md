---
name: "authority-chain-and-gap-detection"
description: "The precedence order between solution document, standards, Jira and repository (solution > standards > jira > repository), and the rules for what counts as a genuine design gap versus something that can be read from the reference/repo or is simply unknown until deployment. Use whenever sources disagree or something needed is absent."
version: 3
created: "2026-08-20"
updated: "2026-08-27"
---

# Authority chain and gap detection

## The chain

```
solution document  >  standards  >  jira  >  repository
```

Higher wins. **`standards`** is `00-inputs/Standards-Profile.json` when a standards document was
supplied for the run; it sits below the solution document and above jira. When no standards document
was supplied the level is simply absent and the chain reads `solution > jira > repository` — so this
ordering is a no-op on a run without one. The **repository** is the lowest authority but the only
source that is *demonstrably real*: its conventions, naming, layout and project skeleton (recorded in
`Repo-Profile.json`, including the named reference project's structure) are what generated code
conforms to unless a higher level overrides. Two refinements that the bare ordering does not capture,
and both matter:

**The solution document is authority for *what to build*. Jira acceptance criteria are
authority for *what "done" means*.** They are not competing on the same question. Where the
solution design describes behaviour and an acceptance criterion describes how that behaviour
will be verified, both stand — the AC is the test, the design is the thing being tested.

**A conflict is only resolved by the chain when the two sources are answering the same
question.** If the solution document says "retry three times" and the repository already has a
shared retry helper, those are compatible: use the helper, configured for three. Reaching for the
chain before checking whether there is a real conflict produces overrides nobody needed.

### Applying it

- Conflict between two levels → the higher level wins, **and you record the override**. State
  what lost, where it came from, and why it lost. A silent override is indistinguishable from
  an oversight when someone reviews the code.
- Conflict **within** the top level — the solution document contradicting itself — has no
  tie-breaker above it. That is always a gap, always for a human.
- An acceptance criterion that cannot be met given the design is a gap, not a design change.
  You do not get to relax the AC and you do not get to redesign around it.
- The repository is bottom of the chain but it is the only source that is *demonstrably real*.
  Where the repository contradicts a higher level, the higher level wins — but say so loudly,
  because a codebase that consistently does something else is evidence that the document is
  stale.

## What is a gap

A gap is a question **a human can answer and the pipeline cannot**.

| Type | What it is | Example |
|---|---|---|
| `AMBIGUITY` | Two readings, both defensible | "the service retries" — how many times, with what backoff? |
| `CONTRADICTION` | Two sources disagree and the chain cannot settle it | solution document §3 says at-least-once, §7 says exactly-once |
| `MISSING` | Something required to proceed is absent | an interface with a request shape and no response shape |
| `OUT_OF_SCOPE` | Something is asked for that this run should not do | an instruction to modify a module the operator excluded |

## What is not a gap

This distinction is where gap detection usually goes wrong, and both directions are costly.

**Not a gap: a value that is unknown until deployment.** An account id, a bucket name, an
endpoint URL, a secret's ARN. These become validated configuration inputs, not questions for
a reviewer. Raising them fills the review file with things nobody can answer, and a reviewer
who has to skim forty of those will miss the three that matter.

**Not a gap: something you could determine by reading.** If the answer is in a file you have
not opened, open it. A gap that a `read_file` would have closed wastes a review round-trip. This
explicitly includes the **reference project** and `Repo-Profile.json`: when the solution names a
reference to model on, anything it already implements — the auth/token flow, configuration and
secret retrieval, retry, a resource it already touches — is available to READ and REUSE, not a
question for a reviewer. Walk the chain for the answer (solution → standards → jira → reference/
repository) before raising a `MISSING` gap; raise it only for what is genuinely absent from every
level. Re-implementing, or gapping, something the reference already provides is the most common
avoidable failure here.

**Not a gap: a decision the specification stage exists to make.** Which module a function
belongs in, what to name a class, how to structure the tests. That is design, not ambiguity.

**Is a gap even though it feels minor: any untestable non-functional requirement.** "fast",
"secure", "scalable". Those become test assertions and alerts, and inventing the number is
how a threshold nobody agreed to ends up in production.

## Severity and the two flags

```json
{
  "id": "GAP-spec-004",
  "type": "CONTRADICTION",
  "authoritySource": "solution",
  "severity": "HIGH",
  "requiresHumanInput": true,
  "blocksCodeGeneration": true,
  "question": "Solution §3 specifies at-least-once delivery; §7 specifies exactly-once. Which applies to the ingest path?",
  "context": "§3.2 'Ingest flow' vs §7.1 'Delivery guarantees'. The two imply different dedupe requirements.",
  "proposedDefault": "at-least-once with idempotency keys, per §3 which is more detailed",
  "resolution": ""
}
```

`requiresHumanInput` — could anything other than a person settle this? If a document you have
not read could, read it instead.

`blocksCodeGeneration` — **can you write correct code without the answer?** If a default
exists that is defensible and reversible, the answer is no: proceed on the default, record it,
and let the reviewer object. Set it `true` only when generating on a guess would produce code
that must be thrown away.

Be sparing with `true`. A blocking gap stops the run; that is right when the alternative is
confidently wrong code, and wrong when a note in the summary would have done.

`proposedDefault` is mandatory. What happens if nobody answers? Every gap needs one, including
blocking ones — it is what the reviewer is agreeing to when they defer.

## Ask a question that can be answered

The reviewer sees your `question` and little else. So:

- **One question per gap.** Two questions in one entry get one answer.
- **Include what they need to decide**, in `context` — the sources, what each says, what turns
  on it. Not the whole document.
- **Ask about the decision, not the mechanism.** "Should a failed message be retried or
  dead-lettered?" is answerable. "How should `_handle_failure` be implemented?" is not.
- **Say what turns on it.** A reviewer who knows the consequence answers faster and better.

## Deduplicate

Several agents read the same documents and will find the same ambiguity. Before adding a gap,
check whether it is already raised. If it is, cite the existing id rather than creating a
second entry — a review file with the same question three times gets one answer and two
deferrals, and it is not obvious afterwards which was which.
