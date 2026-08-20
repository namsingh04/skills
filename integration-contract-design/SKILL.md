---
name: "integration-contract-design"
description: "Specify every boundary the system talks over as request shape, response shape, error behaviour and a concrete sample exchange, and emit the test fixtures that make those contracts testable. Use in the specification stage for interfaces and integrations."
version: 1
created: "2026-08-20"
updated: "2026-08-20"
---

# Integration contract design

A boundary is anywhere this system hands something to, or receives something from, code it
does not control: an HTTP endpoint it serves or calls, a queue it reads or writes, a database
it queries, an event it emits, a file format it exchanges.

Each one gets a contract. A contract without a concrete example is not finished.

## The four parts of a contract

**Request shape** — fields, types, which are required, what validates them.
**Response shape** — the same, for each status or outcome, not just the successful one.
**Error behaviour** — what happens on invalid input, on timeout, on the far side being down.
This is half the contract and it is usually the half that is missing.
**A sample exchange** — real values, taken from the solution document.

Each part cites a requirement or a source. A field you added because it seemed useful is a
field somebody else's system does not send.

## Sample exchanges become the tests

`00-inputs/Solution-Model.json` has a `sampleExchanges` array. That is the most valuable
thing in this pipeline: it is the only place where the expected behaviour exists as *data*
rather than as description, and data is what a test can assert against.

So the samples survive into `20-spec/Test-Fixtures.json` **exactly as written**:

```json
{
  "fixtures": [
    {
      "id": "FIX-002",
      "contract": "CON-001",
      "operation": "POST /ingest",
      "description": "valid message with schema version 2",
      "request": {"headers": {"x-schema-version": "2"}, "body": {"id": "abc-123", "payload": {}}},
      "expectedResponse": {"status": 202, "body": {"accepted": true, "id": "abc-123"}},
      "source": "solution:§4.1 sample request",
      "satisfies": ["FR-004"],
      "kind": "happy-path|error|edge"
    }
  ]
}
```

**Do not normalise a sample.** Do not reformat it, do not add fields that "should" be there,
do not correct what looks like a typo. If a sample is malformed or internally inconsistent,
that is a gap — and quite often it is the real requirement, because the malformed thing is
what the other system actually sends.

**Do not invent samples to fill out coverage.** A fixture you made up tests your assumption,
passes, and proves nothing. Where a contract has no sample, raise a `MISSING` gap and mark the
contract `sampleless: true` so the test author knows to write a structural test rather than a
behavioural one.

Cover the error paths with fixtures too, where the document supplies them. A contract with
five happy-path fixtures and no error fixture will be implemented with no error handling.

## Mismatches are contradictions, not compromises

When two sources describe the same boundary differently — the diagram shows one direction and
the prose another, the sample response has a field the schema does not — record **both** and
raise a `CONTRADICTION` gap.

Never merge them into a shape that satisfies both. That shape matches nothing on either side
of the boundary, and it will fail in integration rather than in test, which is the most
expensive place to find it.

## Direction matters

From `Solution-Model.json`'s `relationships`, an arrow is a dependency direction and becomes a
call direction. Getting it backwards inverts who initiates, who retries, and who is
responsible when the other side is unavailable — and the resulting code is plausible, passes
its own tests, and is completely wrong.

If the arrow direction is unlabelled or ambiguous, that is a gap. It is not something to
infer from which box is drawn on the left.

## Output

Write `20-spec/Integration.json` and `20-spec/Test-Fixtures.json`.

`Integration.json` payload:

```json
{
  "contracts": [
    {
      "id": "CON-001",
      "boundary": "inbound-http|outbound-http|queue|database|event|file",
      "operation": "POST /ingest",
      "direction": "inbound|outbound",
      "counterparty": "upstream ingest client",
      "request": {"fields": [{"name": "", "type": "", "required": true, "validation": ""}]},
      "responses": [{"outcome": "accepted", "status": 202, "fields": []}],
      "errorBehaviour": [
        {"condition": "schema version header absent", "response": "400 with error code SCHEMA_MISSING",
         "requirement": "FR-005"}
      ],
      "idempotency": {"required": true, "key": "message id", "source": ""},
      "timeoutsAndRetries": {"timeoutMs": 5000, "retries": 3, "source": "", "assumed": false},
      "sampleless": false,
      "satisfies": ["FR-004"],
      "source": ""
    }
  ],
  "unresolvedBoundaries": [{"described": "", "why": ""}]
}
```

Mark anything you had to assume with `"assumed": true` and a gap id. Timeouts and retry counts
are the fields most often absent from a design and most often invented by a generator — an
invented retry policy is a production incident waiting for load.

## Status

`OK` — every boundary specified with samples.
`PARTIAL` — specified with gaps for missing samples, directions or error paths. Normal.
`BLOCKED` — the solution model or requirements model was missing or unparseable.
