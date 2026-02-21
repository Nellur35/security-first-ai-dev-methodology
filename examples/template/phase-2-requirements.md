# Phase 2 — Product Requirements

**Input:** Phase 1 problem statement

## Functional Requirements

- [ ] [The system must do X. Testable: verify by Y.]
- [ ] [The system must do Z. Testable: verify by W.]
- [ ] ...

**Example:**

- [ ] The system must assess the security posture of an AWS account in under 60 seconds. *Testable: time the assessment Lambda and assert < 60s.*
- [ ] The system must deploy security configurations via CloudFormation, not direct API calls. *Testable: verify no direct API mutations in deploy code path.*
- [ ] All deployed resources must be tagged with `ManagedBy`, `RecipeId`, and `RecipeVersion`. *Testable: assert tags on every resource in the stack.*

## Non-Functional Requirements

- [ ] [Performance: e.g., API response time < 500ms at p99]
- [ ] [Security: e.g., All secrets from environment variables or parameter store, never hardcoded]
- [ ] [Reliability: e.g., Single component failure must not cascade]

## Explicit Exclusions — What This System Does NOT Do

- [e.g., Does NOT support multi-cloud]
- [e.g., Does NOT provide real-time monitoring]
- [e.g., Does NOT replace existing SIEM tools]

## Definition of Done

[What does success look like in reality — not on a dashboard?]

**Example:**

> A new user can launch the product, run an assessment, deploy a security baseline, and verify compliance — all without reading documentation beyond the initial setup guide. The full pipeline passes. No manual steps required after initial deployment.

## Decisions & Rejected Alternatives

| Decision | Alternative Rejected | Reason |
|----------|---------------------|--------|
| [e.g., YAML for configuration] | [e.g., JSON] | [e.g., Human readability for security recipes matters more than parsing speed] |
| [e.g., CloudFormation for deployment] | [e.g., Direct API calls] | [e.g., CFN provides rollback, drift detection, and audit trail] |

---

*This file is the sole input to Phase 3. Everything not written here does not carry forward.*
