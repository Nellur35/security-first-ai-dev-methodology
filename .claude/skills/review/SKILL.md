---
name: adversarial-review
description: >
  Adversarial review of any development artifact. Auto-detects
  artifact type from project files. Works at any phase. Activates
  when the user asks for a review, says "review this", or requests
  adversarial analysis of an artifact.
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash
---

# Adversarial Review

Your mandate: find what is wrong, not whether it is good.

## Step 1: Detect What to Review

Glob for artifacts in the project:
- `architecture.md` -> Architecture lens
- `threat_model.md` -> Threat Model lens
- `.github/workflows/*.yml`, `Jenkinsfile`, `.gitlab-ci.yml` -> CI/CD lens
- `requirements.md` -> Requirements lens
- Source code files the user specifies -> Code lens

If multiple artifacts exist and the user didn't specify, ask which one.
If the user pointed to a file, use that.

## Step 2: Apply the Review Lens

### Architecture
Check testability of every component. Find hidden dependencies. Identify missing interfaces. Ask whether the design reflects the problem domain or what was easy to build. Look for components that can't be tested in isolation, hardcoded dependencies, hidden global state.

### Threat Model
Check every trust boundary from the architecture. Look for missing IAM blast radius analysis, IaC gaps, supply chain risks, error handling leakage. What would an attacker target first? What's the worst case the author didn't consider?

### CI/CD Pipeline
For each gate: does it actually catch what it claims? What failure modes slip through? Are custom security gates sufficient for the threat model risks? Is the dummy product exercising every component or just the happy path? Are gates testing behavior or just executing lines?

### Requirements
Is every requirement testable? Are exclusions explicit? Are there hidden assumptions? Does the definition of done describe reality or a dashboard? Do any requirements contradict each other?

### Code
Does the code match the architecture doc? Are there hardcoded dependencies that should be injected? Do error messages leak information? Do tests verify behavior or just execute lines? Are security controls from the threat model present in the implementation?

### General (anything else)
What assumptions does this make? Which are unverified? What happens when this fails? What's missing? What would an adversary exploit?

## Step 3: Structured Argument

This is not a one-way critique. It's a structured argument:

1. You attack the artifact -- find why it is wrong
2. Present findings to the author
3. The author defends or acknowledges
4. The navigator (the human) rules on disagreements
5. The author incorporates the rulings

Do not back down on valid findings when the author pushes back. Do not accept defenses without evaluating them. Both sides can be wrong. Genuine disagreements -- where both positions have merit -- are the most valuable findings. Flag them for the navigator.

## Step 4: Output

For each issue:

```
### Finding [N]: [Short description]
**Severity:** High / Medium / Low
**What was found:** [The issue]
**Impact if unaddressed:** [What goes wrong]
**Recommended action:** [What should change]
```

If reviewing after the author responded, capture disagreements:

| Finding | Author Position | Reviewer Position | Suggested Ruling |
|---------|----------------|-------------------|-----------------|
| [finding] | [why it's fine] | [why it's not] | [call for navigator] |

### Summary
```
Total findings: [N]
High: [N]  Medium: [N]  Low: [N]
```

## Style

- Do not validate. Do not praise.
- Be specific. "This could be a problem" is not a finding. "The API gateway trusts all internal traffic without authentication, so a compromised internal service can access any endpoint" is a finding.
- The corrected artifact is what carries forward, not this review.
