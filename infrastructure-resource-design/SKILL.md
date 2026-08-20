---
name: "infrastructure-resource-design"
description: "Turn requirements and the tables that accompany them - IAM policies, VPC and network settings, resource configuration - into a resource specification where every resource, permission and boundary traces to a source row. Use in the specification stage for infrastructure work."
version: 1
created: "2026-08-20"
updated: "2026-08-20"
---

# Infrastructure resource design

You are turning tables and requirements into a resource specification. The tables are the
authoritative part: they contain exact values that appear nowhere else, and a value you
paraphrase is a value that is now wrong.

This skill is provider-neutral. Whether the target is AWS, Azure, GCP, Kubernetes or a
docker-compose file, the discipline is the same: exact values, traced permissions, explicit
boundaries.

## Tables are transcribed, never summarised

From `00-inputs/Solution-Model.json`, the `tables` array holds what was extracted.

- Copy values **exactly**. `10.0.0.0/16` is not "the VPC range". `s3:GetObject` is not "read
  access". An ARN with a wildcard is not the same as one without.
- Keep every column. A column that looked irrelevant during extraction is a condition key, a
  region, or an environment qualifier, and it cannot be recovered later.
- A row you cannot map to a resource is a gap. Do not drop it — an unmapped IAM row usually
  means a resource nobody specified.

## Every permission traces to a requirement or a row

For each permission in your output:

```json
{
  "principal": "ingest-lambda-role",
  "actions": ["s3:GetObject"],
  "resources": ["arn:aws:s3:::ingest-bucket/*"],
  "conditions": {},
  "source": "solution:table 'IAM policies' row 4",
  "justification": "FR-004 requires reading the raw message from the landing bucket"
}
```

A permission with no `source` and no `justification` does not go in the spec. That is not
pedantry: unattributed permissions are how least privilege quietly stops being least, and
nobody reviewing the generated code can tell which ones were asked for.

## Over-broad access is never the safe default

If the tables do not specify the scope of a permission, **raise a gap**. Do not write `*`.

`Resource: "*"` and `Action: "s3:*"` are the two most consequential things a code generator
can invent. They pass every test, satisfy every requirement, and are found much later by
someone else.

Where you must proceed without an answer, propose the **narrowest** default that could satisfy
the requirement, mark it as an assumption, and set the gap's `proposedDefault` to it. Narrow
and wrong fails loudly in testing; broad and wrong does not fail at all.

## Network boundaries are explicit

For anything with network placement: which subnet tier, which security group, what ingress,
what egress, and what is deliberately *not* reachable.

State the negatives. "This function has no internet egress" is a design decision that will be
silently reversed by whoever adds the next NAT route unless it is written down as intentional.

## What you produce

Write `20-spec/Infrastructure.json`. In `payload`:

```json
{
  "provider": "aws|azure|gcp|kubernetes|other|none",
  "resources": [
    {
      "id": "RES-003",
      "type": "queue",
      "logicalName": "ingest-dlq",
      "purpose": "",
      "configuration": {"retentionSeconds": 1209600},
      "source": "solution:table 'Queues' row 2",
      "satisfies": ["FR-004"],
      "dependsOn": ["RES-001"]
    }
  ],
  "permissions": [ … ],
  "network": {
    "vpc": {"cidr": "", "source": ""},
    "subnets": [{"name": "", "cidr": "", "tier": "public|private|isolated", "source": ""}],
    "securityGroups": [{"name": "", "ingress": [], "egress": [], "source": ""}]
  },
  "configurationInputs": [
    {"name": "INGEST_BUCKET", "kind": "deploy-time", "required": true, "default": null,
     "note": "unknown until deployment; validated at startup, not a design gap"}
  ],
  "assumptions": [{"statement": "", "because": "", "gap": ""}],
  "unmappedTableRows": [{"table": "", "row": 7, "why": ""}]
}
```

`configurationInputs` is where deployment-time unknowns go — account ids, bucket names,
endpoints, secret references. **These are not design gaps.** Recording them here as validated
configuration is the correct handling; putting them in the review file fills it with questions
nobody can answer and buries the ones that matter.

`unmappedTableRows` is the honest counterpart: rows you could not place. Each one is a gap.

## Coupling to the code stage

Your `logicalName` values and `configurationInputs` names are what the code agents will use
verbatim. Choose them once, per the standards profile's naming rules, and do not restate the
same resource under two names in different parts of the file — a validator will catch it, but
only after a retry that could have been avoided.

## Status

`OK` — every table row mapped, every permission traced.
`PARTIAL` — mapped with gaps for unscoped permissions or unmapped rows. Normal.
`BLOCKED` — the solution model or requirements model was missing or unparseable.
