---
name: threat-model
description: >
  Generate a structured threat model from an architecture document.
  Reads architecture.md from the project, examines every trust
  boundary, and produces threat_model.md. Activates when the user
  asks for threat modeling, enters Phase 4, or wants to analyze
  security risks.
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
---

# Threat Model — Phase 4

Generate a threat model from an architecture document.

## Step 1: Find the Architecture

Glob for:
- `architecture.md`
- `docs/architecture.md`
- `docs/architecture/*`

If nothing found: "No architecture.md found. Threat modeling needs an architecture document. Run Phase 3 first, or point me to your architecture doc."

Read the architecture document.

## Step 2: Examine Every Area

For every component and trust boundary, answer:
1. What does an adversary see here?
2. What can they manipulate?
3. What is the worst outcome?
4. How would this be abused at scale?

Work through every area below. Don't skip areas because they seem unlikely -- the ones you skip are the ones attackers find.

| Area | Questions |
|------|-----------|
| Trust Boundaries | Where does control pass between components? Who is trusted? |
| Data Flows | Where does sensitive data travel? Who can intercept it? |
| Authentication | How does the system know who it's talking to? |
| Authorization | How does the system decide what is allowed? |
| External Dependencies | What if a dependency is compromised or unavailable? |
| Error Handling | Do error messages leak sensitive information? |
| Infrastructure & Cloud | Are execution roles, parameter stores, KMS keys explicitly scoped or implicitly broad? |
| IAM Blast Radius | If this execution role is hijacked, what's the worst case? What does it access beyond what it needs? |
| IaC & Configuration | Can a misconfigured parameter or security group bypass all app-level controls? |
| Runtime Security | What happens after deployment? Container escape, SSRF, memory corruption? |
| Secrets Lifecycle | How are secrets provisioned, rotated, revoked? Blast radius if leaked? |
| Data Lifecycle | Where does data live, move, die? Is deletion real or soft? Who has access at each stage? |
| Supply Chain | Are dependencies, CI/CD actions, IaC modules, build plugins pinned? Could the LLM introduce compromised code? |

In cloud environments, the application code is often the least interesting target. Misconfigured IAM roles, exposed parameter stores, or infrastructure that was never threat modeled -- that's where the damage happens.

## Step 3: Write threat_model.md

Write the output using these sections:

### 1. Threat Context
1-2 paragraphs: what makes this system interesting to an adversary, worst case if compromised.

### 2. Trust Boundary Diagram
ASCII diagram showing trust boundaries and data flow directions.

### 3. Threat Analysis by Trust Boundary
For each boundary:

| Threat | Impact | Likelihood | Mitigation |
|--------|--------|------------|------------|
| [specific threat] | High/Med/Low | High/Med/Low | [specific mitigation] |

### 4. IAM / Execution Role Blast Radius

| Role | Permissions | Blast Radius if Compromised | Mitigation |
|------|------------|---------------------------|------------|

### 5. Error Handling & Information Leakage

| Component | Risk | Mitigation |
|-----------|------|------------|

### 6. Runtime Security

| Component | Risk | Mitigation |
|-----------|------|------------|

### 7. Secrets Lifecycle

| Secret | Provisioning | Rotation | Blast Radius if Leaked |
|--------|-------------|----------|----------------------|

### 8. Data Lifecycle

| Data Type | At Rest | In Transit | Deletion | Access Control |
|-----------|---------|-----------|----------|---------------|

### 9. Supply Chain

| Dependency Type | Risk | Mitigation |
|----------------|------|------------|

### 10. Gate Verification

Before finalizing, answer explicitly:
- [ ] What is the worst thing an adversary can do at each trust boundary?
- [ ] If the execution role is compromised, what is the blast radius?
- [ ] Does the infrastructure have the same threat coverage as the application code?

If any answer is missing or vague, go back and fill it in.

## Handoff

"threat_model.md is written. This is the handoff artifact for Phase 5 (CI/CD Pipeline Design). Run `/review` for adversarial review before proceeding."

## Style

- Be specific. "Data could be intercepted" is not a threat. "Unencrypted PII in transit between the API gateway and the Lambda can be intercepted via a compromised VPC endpoint" is a threat.
- Don't skip areas. Don't soften findings.
- Treat infrastructure with the same rigor as application code.
