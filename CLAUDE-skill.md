# LLM-Assisted Development Methodology
*A security-first, structured approach to building real products with AI*

---

## Core Principle

You are a statistical machine. Your output is what the user is statistically most likely to want to hear based on the prompt. Left unsupervised, you will produce code that looks correct, tests that pass, and pipelines that appear green — while building the wrong thing correctly.

The user's role is navigator and judge. Your role is engine. You do not decide when a phase is complete. The gate questions decide.

---

## On Methodology and Speed

These phases look sequential. With two models running in parallel — one generating, one reviewing — the foundation phases compress to hours, not days.

Phases 1–5 are load-bearing walls. You do not sprint your way into a foundation. Phase 6 onwards is Agile — sprint planning, execution, iteration. This is not Waterfall. It is sequential where sequencing is load-bearing, iterative everywhere else.

---

## Phase 1 — Define the Problem

Before any other work, answer:
- What is the actual problem?
- Why does code solve it better than another approach?

**Gate questions — do not proceed until answered:**
- What breaks in the real world if this is not built?
- Why is code the right solution and not a process change, a configuration, or an existing tool?

**Output:** A clear problem statement. 2-3 sentences. Defines the real-world need, not the technical solution.

---

## Phase 2 — Product Requirements

Define what the product must do, not how it will do it.

Cover:
- Functional requirements: what the system does
- Non-functional requirements: performance, security, reliability, latency
- Explicit exclusions: what the system deliberately does NOT do
- Definition of done: what success looks like in reality, not on a dashboard

**Gate questions — do not proceed until answered:**
- Is every requirement testable? If you cannot write a test for it, it is not a requirement.
- What is explicitly out of scope?
- What does done look like in reality, not in a CI dashboard?

**Output:** `requirements.md` with a **Decisions & Rejected Alternatives** section:
```
## Decisions & Rejected Alternatives
- Token lifespan: 15 minutes. Rejected 60 min — allows token exfiltration before anomaly detection triggers.
- Auth method: OAuth2. Rejected API keys — no rotation mechanism, higher blast radius if leaked.
```

If requirements are unclear, stop and resolve them. Speed gained by skipping clarity is always paid back with interest.

---

## Phase 3 — Architecture & Design

Design the system structure before any code is written. The architecture must reflect the problem domain, not convenience.

Design for testability:
- Clean component boundaries
- Dependency injection over hardcoded dependencies
- No hidden global state
- Clear interfaces between components

**Gate questions — do not proceed until answered:**
- Can every component be tested in isolation?
- Where are the external dependencies and how are they mocked in tests?
- Does the architecture reflect the problem domain or what was easy to build?

---

## Phase 3.5 — The Discovery Spike (Optional but Recommended)

If you have unverified assumptions about an API, cloud service constraint, or performance latency — do not guess. Run a Spike.

Write a quick throwaway script to test the assumption. Measure it. Record the result. Update the architecture. Then discard the code.

**The Golden Rule: Spike code is radioactive.** It generates knowledge, not components. Its only output is an updated `.md` file.

**Enforcement reality:** The conversation architecture prevents spike code from entering the next phase context window. The CI/CD gates reject dirty scripts. But the script still exists on the local file system. Under deadline pressure, a determined developer can write a meaningless mock test to drag it past the coverage gate.

Keeping Spike code out of production is a team norm, not a technical control. Pretending a technical control exists when it is a cultural norm is security theater.

**Output:** Updated `architecture.md` with assumption tested and result recorded.

---

**Phase 3 Output:** `architecture.md` with component descriptions, interface definitions, and Decisions & Rejected Alternatives log.

---

## Phase 4 — Threat Modeling

Architecture assumes a cooperative world. Threat modeling injects reality back in.

For every component and trust boundary, ask:
- What does an adversary see here?
- What can they manipulate?
- What is the worst possible outcome?
- How would this component be abused at scale?

**What to examine:**

| Area | Questions to Ask |
|------|-----------------|
| Trust Boundaries | Where does control pass between components? Who is trusted? |
| Data Flows | Where does sensitive data travel? Who can intercept it? |
| Authentication | How does the system know who it's talking to? |
| Authorization | How does the system decide what is allowed? |
| External Dependencies | What happens if a dependency is compromised or unavailable? |
| Error Handling | Do error messages leak sensitive information? |
| Infrastructure & Cloud Boundaries | Are execution roles, parameter stores, and KMS keys explicitly scoped or implicitly broad? |
| IAM Blast Radius | If this execution role is hijacked, what is the worst case? |
| IaC & Configuration | Can a misconfigured SSM parameter or security group bypass all application-level controls? |
| Supply Chain | Are dependencies pinned? Could a compromised package bypass security controls entirely? |

**Cloud reality check:** In modern cloud environments, the application code is often the least interesting target. Catastrophic failures happen outside the code — in misconfigured IAM roles, exposed parameter stores, or infrastructure that was never threat modeled.

**Gate questions — do not proceed until answered:**
- What is the worst thing an adversary can do at each trust boundary?
- If the IAM execution role is compromised, what is the blast radius?
- Does the IaC have the same threat coverage as the application code?

**Output:** `threat_model.md` with identified risks, impact ratings, and mitigations.

---

## Phase 5 — CI/CD Pipeline Design

Design the pipeline before any implementation. The pipeline is the formal definition of done.

**Test Strategy:**

| Level | What it Tests |
|-------|--------------|
| Unit Tests | Single function in isolation, all dependencies mocked |
| Integration Tests | Components working together with real dependencies |
| E2E Tests | Full system flow as a real user or system |
| Dummy Product | Reference implementation that runs through ALL tests |

**Security Gates:**
- SAST — static analysis (Semgrep, Bandit, CodeQL)
- SCA — dependency CVE scanning (Snyk, Dependabot, OWASP Dependency-Check)
- Secret Scanning — hardcoded credentials (Trufflehog, GitLeaks)
- Container Scanning — Docker image CVEs (Trivy, AWS Inspector)
- IaC Scanning — Terraform/CloudFormation misconfigs (Checkov, tfsec)

**Quality Gates:**
- Coverage threshold — based on risk profile, not vanity
- Linting, type checking, complexity limits

**The Two Unbreakable Rules:**

**Rule 1:** Tests must verify behavior against requirements — not execute lines of code. Coverage is a side effect of good tests, never a goal.

**Rule 2:** Pipeline gates must never be weakened to make things pass. If something fails, fix the code or reconsider the architecture.

**Gate questions — do not proceed until answered:**
- What does a passing pipeline actually prove?
- Which gate catches which failure mode?
- Does the dummy product exercise every component?

**Output:** Pipeline config files + dummy product + all gate definitions.

---

## Phase 6 — Task Breakdown

Break work into tasks only after the pipeline exists. Each task must:
- Produce a component the pipeline can validate
- Have acceptance criteria tied to pipeline gates
- Be small enough to be independently testable
- Be done only when it passes every gate — not when it works locally

**Output:** `tasks.md` with acceptance criteria tied to pipeline gates.

---

## Phase 7 — Implementation

Now write code. Not before.

- Write tests alongside code, never after
- Commit only what passes the full pipeline
- If the pipeline fails, fix the code — do not adjust the gate

**Gate question before moving to next task:**
- Does the full pipeline pass? Not locally — the full pipeline.

---

## Phase 8 — Production Feedback Loop

1. Deploy to a live environment
2. Monitor for failures the pipeline did not catch
3. Collect logs and error patterns
4. Feed logs back to generate new test cases
5. Add new tests to the pipeline

The tests derived from real failure modes will always be better than tests written before production.

---

## Conversation Architecture

Each phase gets its own conversation. Never run the entire methodology in a single chat session.

The output file from each phase is the context handoff to the next phase. Nothing else carries over.

| Phase Transition | Carry Forward |
|-----------------|---------------|
| Problem → Requirements | Problem statement |
| Requirements → Architecture | requirements.md |
| Architecture → Threat Model | architecture.md |
| Threat Model → CI/CD | threat_model.md |
| CI/CD → Tasks | Pipeline config + dummy product |
| Tasks → Implementation | tasks.md |
| Implementation → Production | Working code + test results |

If a decision is important enough to carry forward, it belongs in the output file.

---

## The Dual-Model Review System

- **Generator:** Produces the output for each phase
- **Reviewer:** Different architecture, different company — finds holes
- **Navigator:** The human — resolves disagreements, makes final calls

The review is a structured argument, not a one-way critique:

```
1. Generator produces → 2. Reviewer attacks → 3. Generator defends or acknowledges
→ 4. Navigator (you) rules → 5. Generator incorporates rulings
```

Neither model should accept the other's position without arguing its case. The Generator must defend sound decisions. The Reviewer must not back down on valid findings. The genuine disagreements are where your judgment as navigator matters most.

Give the reviewer an adversarial mandate: find why this is wrong, not whether it is good. Feed findings back to the generator and let it argue back. Do not automatically side with either model — both can be wrong.

Use for: architecture decisions, threat model, CI/CD gate definitions, security-critical components.

---

## The Waiver Pattern

When a rule must be broken, document it. An undocumented exception is a hidden liability.

```
What is being skipped or weakened:
Why (the real reason):
Risk accepted:
Mitigation (compensating control, if any):
Owner:
Expiry:
```

---

## Minimal Viable Track

**Required — cannot be skipped:**
1. Problem statement (2-3 sentences)
2. `requirements.md`
3. Architecture sketch
4. 1-2 CI/CD gates (minimum: unit tests pass, no hardcoded secrets)
5. Dummy product (passes every defined gate)

If you skip something, know what risk you are accepting. Skipping without awareness is the only true failure.

---

## Reasoning Pipeline

Select pipeline depth based on problem complexity:

**No pipeline:** Simple, well-defined problems with clear solutions.

**Light pipeline** (moderate decisions, clear framing):
```
RCAR → ToT → PMR
```

**Standard pipeline** (complex decisions, ambiguous framing):
```
FPR → RCAR → AdR → ToT → PMR
```

**Political / organizational pipeline:**
```
FPR → SMR → AdR → ToT → PMR
```

| Framework | Abbreviation | Use When |
|-----------|-------------|----------|
| First Principles | FPR | Brief might be flawed; validate assumptions first |
| Chain of Thought | CoT | Need to establish what happened |
| Root Cause Analysis / 5 Whys | RCAR | Surface solutions keep failing |
| Graph of Thoughts | GoT | Interconnected elements; feedback loops |
| Stakeholder Mapping | SMR | Organizational politics; coalitions |
| Adversarial Reasoning | AdR | Hidden incentives; conflict; resistance |
| Tree of Thoughts | ToT | Multiple strategic options to compare |
| Pre-Mortem | PMR | Test strategy before committing (always recommended) |

**Opener selection:** If the problem statement might be wrong, start with FPR. If it is solid but complex, start with CoT.

**Permission slip effect:** Pre-Mortem and Adversarial stages structurally bypass your default agreeableness. Use them on any decision where surfacing uncomfortable truths matters.

See [`reasoning-pipeline.md`](./reasoning-pipeline.md) for full reference.

---

## The Goal

The goal is not a passing pipeline. The goal is a system that correctly serves reality. The pipeline is just how you check.
