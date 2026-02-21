# Phase 1 — Problem Definition: CloudJanitor Brain Agent

## Problem Statement

CloudJanitor Phase 1 automated the deployment of AWS security configurations through curated recipes and Lambda tools. But customers still need to decide which recipes to deploy, in what order, and why — a task requiring both AWS security expertise and understanding of the recipe catalog. Without a reasoning layer, customers face a cold-start problem: they have powerful tools but no guidance on how to use them for their specific situation.

## Gate Questions

### What breaks in the real world if this is not built?

- Security teams receive compliance audit findings (SOC2, ISO27001, CIS) but don't know which CloudJanitor recipes address which findings
- New AWS accounts remain unsecured for days while teams research configuration steps
- Organizations deploy recipes in wrong order (e.g., monitoring before CloudTrail, creating alarm noise)
- Security engineers spend 4-6 hours per account researching which controls to deploy instead of investigating actual threats
- CloudJanitor adoption stays low because it requires too much upfront knowledge to use effectively

### Why is code the right solution and not a process change, a configuration, or an existing tool?

**Why not process documentation:** Decision trees and guides become stale as AWS services evolve and new recipes are added. Humans don't execute multi-stage reasoning consistently under deadline pressure.

**Why not static rules:** If/then rules can't handle ambiguity. "Improve our security posture" vs "pass SOC2 audit by Q2" vs "secure this specific workload" require different reasoning strategies. Rule-based systems also can't explain their reasoning — customers need transparency.

**Why not existing tools:** AWS Security Hub aggregates findings but doesn't deploy fixes. Compliance scanners (Prowler, CloudMapper) generate reports but don't map to executable recipes. Generic LLM access lacks domain knowledge, Lambda access, and validation guardrails.

**Why an AI reasoning layer:** This is a knowledge synthesis problem — given current state, goals, constraints, and compliance requirements, what is the optimal sequence of actions with acceptable risk and cost? That requires structured reasoning over dynamic data. Code can enforce safety rails, provide reasoning traces, and adapt depth to request complexity.

**Honest caveat:** With only 5 curated recipes today, a well-designed questionnaire could handle most routing. The LLM layer's primary value at this scale is natural language understanding, reasoning transparency, and Pre-Mortem warnings that a decision tree cannot produce. As the recipe catalog grows (24 planned), the reasoning complexity will justify the LLM layer on its own merits.

## Decisions & Rejected Alternatives

| Alternative | Why Rejected |
|-------------|-------------|
| Static decision tree documentation | Cannot adapt to environment state; becomes stale; cannot produce reasoning traces or deployment warnings |
| Rule-based expert system | Cannot handle ambiguity; brittle when recipes change; no reasoning transparency |
| AWS Security Hub native recommendations | Detects problems but doesn't deploy solutions; no CloudJanitor integration |
| Generic ChatGPT/Claude access | No access to assess_environment; no recipe domain knowledge; no validation guardrails |
| Hardcoded recipe dependency graph | Can encode ordering but can't reason about tradeoffs, cost, or customer context |
