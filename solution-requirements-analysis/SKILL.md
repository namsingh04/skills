---

name: "solution-requirements-analysis"
description: "Requirements analysis guidelines for transforming Solution Design Documents and Jira Acceptance Criteria into a canonical, traceable infrastructure requirements specification for downstream AWS architecture planning and CDK implementation"
version: 1
created: "2026-07-24"
updated: "2026-07-24"
---------------------

## When to Use

Use when analyzing a Solution Design Document and Jira Acceptance Criteria as the first stage of an AWS CDK infrastructure generation workflow. Apply before architecture planning or CDK implementation to identify required infrastructure, configurations, dependencies, constraints, acceptance criteria, and requirement gaps.

The output of this skill is consumed by the downstream Architecture Planner and MUST describe what the solution requires without making final architecture or CDK implementation decisions.

## Procedure

1. **Solution Understanding**: Analyze the Solution Design Document to understand the proposed solution, scope, functional requirements, infrastructure requirements, existing infrastructure, and stated constraints.

2. **Acceptance Criteria Extraction**: Extract all acceptance criteria from the Jira template. Preserve the original acceptance criteria identifiers and descriptions where available.

3. **Infrastructure Requirements**: Identify all explicitly required infrastructure resources, including AWS services, resource types, configurations, properties, and behaviors stated in the source documents.

4. **Environment & Region Requirements**: Extract all explicitly stated environments, AWS accounts, regions, deployment stages, and environment-specific requirements.

5. **Resource Relationships**: Identify explicitly stated relationships and dependencies between resources, such as event sources, targets, queues, tables, APIs, functions, and other infrastructure components.

6. **IAM & Security Requirements**: Extract explicitly stated IAM, permissions, encryption, authentication, authorization, secrets, security, and compliance requirements.

7. **Networking Requirements**: Extract explicitly stated VPC, subnet, security group, endpoint, routing, connectivity, and network isolation requirements.

8. **Configuration Requirements**: Identify configuration values, environment-specific parameters, feature flags, resource provisioning flags, naming requirements, and other configuration requirements.

9. **Constraints & Dependencies**: Extract technical constraints, external dependencies, integration requirements, technology restrictions, deployment constraints, and dependencies on existing infrastructure.

10. **Acceptance Criteria Mapping**: Map each acceptance criterion to one or more related requirements, resources, or infrastructure components when the relationship is explicitly supported by the documents.

11. **Source Traceability**: Preserve source references for important requirements and acceptance criteria. Clearly identify whether information originated from the Solution Design Document or Jira Acceptance Criteria.

12. **Gap Detection**: Identify information required for downstream architecture planning that is missing, ambiguous, conflicting, unsupported, or unclear.

13. **Gap Classification**: Classify each gap using:

    * `MISSING_INFORMATION`
    * `AMBIGUOUS_REQUIREMENT`
    * `CONFLICTING_REQUIREMENT`
    * `UNSUPPORTED_REQUIREMENT`

14. **Gap Severity**: Assign one severity:

    * `CRITICAL`
    * `HIGH`
    * `MEDIUM`
    * `LOW`

15. **Gap Action**: Assign one action:

    * `NEEDS_CLARIFICATION`
    * `CAN_PROCEED_WITH_DEFAULT`
    * `REQUIRES_ARCHITECTURE_DECISION`

16. **Analysis Status**: Set the overall analysis status:

    * `COMPLETE` when no blocking gaps exist.
    * `GAPS_FOUND` when gaps exist but architecture planning can continue.
    * `CLARIFICATION_REQUIRED` when one or more blocking gaps require clarification before architecture planning.

17. **Canonical Output**: Return the extracted requirements, acceptance criteria, relationships, constraints, traceability information, and gaps using the configured output schema.

## Pitfalls

* NEVER invent requirements that are not supported by the Solution Design or Jira Acceptance Criteria.
* Do NOT assume AWS services, resource configurations, regions, environments, or deployment patterns that are not specified.
* Do NOT make final architecture decisions.
* Do NOT determine CDK stack boundaries.
* Do NOT select CDK constructs or implementation patterns.
* Do NOT generate CDK code.
* Do NOT use the CDK Standard.md file to infer client requirements.
* Do NOT treat organizational CDK standards as project requirements.
* Do NOT silently resolve conflicting requirements.
* NEVER discard ambiguous or incomplete requirements; record them as gaps.
* Do NOT mark a requirement as complete when the source documents do not provide sufficient information.
* Do NOT lose acceptance criteria identifiers or source references when they are available.
* Do NOT claim that an acceptance criterion is satisfied unless the source documents explicitly support the requirement.
* Do NOT convert assumptions into requirements.
* Do NOT merge unrelated requirements into a single requirement.
* Do NOT omit the `gaps` array when no gaps are found; return an empty array instead.
* Do NOT generate architecture recommendations as part of the requirements analysis.

## Verification

1. Verify both the Solution Design Document and Jira Acceptance Criteria were analyzed.
2. Confirm all identifiable infrastructure resources are captured.
3. Confirm resource configurations explicitly stated in the source documents are preserved.
4. Verify all environments and regions explicitly specified in the documents are captured.
5. Verify resource relationships and dependencies explicitly stated in the documents are captured.
6. Confirm IAM and security requirements are captured when specified.
7. Confirm networking requirements are captured when specified.
8. Verify configuration and environment-specific requirements are captured.
9. Confirm all acceptance criteria are extracted and preserved.
10. Verify acceptance criteria are mapped to related requirements or resources where supported.
11. Confirm source traceability is preserved for requirements and acceptance criteria.
12. Verify missing information is recorded as a gap rather than assumed.
13. Verify conflicting requirements are recorded as gaps rather than silently reconciled.
14. Confirm every gap has a type, severity, action, description, and source reference when available.
15. Verify `analysisStatus` accurately reflects whether blocking gaps exist.
16. Confirm the output contains no CDK code or final architecture decisions.
17. Confirm the output conforms to the configured structured Output Schema.
