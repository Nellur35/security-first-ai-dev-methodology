---
name: security-first-methodology
description: >
  Security-first development methodology for AI-assisted coding.
  8-phase workflow with threat modeling, conversation architecture,
  and cross-model adversarial review. Activates when the user starts
  a new project, asks about development methodology, or needs to
  structure AI-assisted development work.
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
  - WebSearch
  - WebFetch
  - Task
---

# Security-First AI Dev Methodology

The user is the navigator and judge. You are the engine. You do not decide when a phase is complete. The gate questions decide. Phases 1-5 are sequential and non-negotiable. Phase 6 onwards is iterative.

### Phase 1 — Define the Problem
Answer: What is the actual problem? Why does code solve it better than another approach?

**Gate — do not proceed until answered:**
- What breaks in the real world if this is not built?
- Why is code the right solution and not a process change, a configuration, or an existing tool?

**Output:** Problem statement (2-3 sentences). **Handoff artifact:** problem statement → Phase 2.

### Phase 2 — Product Requirements
Define what the product must do, not how it will do it.

**Gate — do not proceed until answered:**
- Is every requirement testable?
- What is explicitly out of scope?
- What does done look like in reality, not on a dashboard?

**Output:** `requirements.md` with Decisions & Rejected Alternatives section. **Handoff artifact:** `requirements.md` → Phase 3.

### Phase 3 — Architecture & Design
Design for testability: clean boundaries, dependency injection, no global state.

**Gate — do not proceed until answered:**
- Can every component be tested in isolation?
- Where are the external dependencies and how are they mocked?
- Does the architecture reflect the problem domain or what was easy to build?

**Output:** `architecture.md` with component diagram, interfaces, and Decisions & Rejected Alternatives. **Handoff artifact:** `architecture.md` → Phase 4. Offer adversarial review prompt for a different model.

### Phase 4 — Threat Modeling
For every component and trust boundary, ask: What does an adversary see? What can they manipulate? What is the worst outcome?

Examine: trust boundaries, data flows, authentication, authorization, external dependencies, error handling, IAM blast radius, IaC configuration, supply chain, runtime security, secrets lifecycle, data lifecycle.

**Gate — do not proceed until answered:**
- What is the worst thing an adversary can do at each trust boundary?
- If the IAM execution role is compromised, what is the blast radius?
- Does the IaC have the same threat coverage as the application code?

**Output:** `threat_model.md` with risks, impact ratings, and mitigations. **Handoff artifact:** `threat_model.md` → Phase 5. Offer adversarial review prompt for a different model.

### Phase 5 — CI/CD Pipeline Design
The pipeline is the formal definition of done. Design it before implementation.

Include: unit tests, integration tests, E2E tests, dummy product, coverage threshold, linting, type checking. For security gates (SAST, SCA, secret scanning, container scanning, IaC scanning), select tools based on the architecture (Phase 3) and threat model (Phase 4) — do not use a generic list.

**Two unbreakable rules:**
1. Tests verify behavior against requirements — not execute lines of code.
2. Pipeline gates are never weakened to make things pass.

**Gate — do not proceed until answered:**
- What does a passing pipeline actually prove?
- Which gate catches which failure mode?
- Does the dummy product exercise every component?

**Output:** Pipeline config + dummy product + gate definitions. **Handoff artifacts:** pipeline config + dummy product + `requirements.md` + `threat_model.md` → Phase 6. Offer adversarial review prompt for a different model.

### Phase 6 — Task Breakdown
Each task must: produce a pipeline-validatable component, have acceptance criteria tied to pipeline gates, be independently testable, be done only when it passes every gate.

Phase 6 takes multiple inputs — task acceptance criteria must trace back to requirements and threat model risks.

Task format: `### Task [ID]: [Component] / Files: [paths] / Dependencies: [prior tasks] / Acceptance criteria: [behavior from requirements.md verified by specific test, security gate mapped to threat_model.md risk] / Pipeline gates exercised: [list]`

**Output:** `tasks.md` with acceptance criteria tied to pipeline gates. **Handoff artifact:** `tasks.md` → Phase 7.

### Phase 7 — Implementation
Write tests alongside code. Commit only what passes the full pipeline. If the pipeline fails, fix the code — do not adjust the gate. Before moving to next task: all acceptance criteria checked, full pipeline passes, no new regressions.

**Handoff artifact:** working code + test results → Phase 8.

### Phase 8 — Production Feedback Loop
Deploy, monitor, collect failure patterns, generate new tests, feed them back into the pipeline. For each finding, capture: what happened, what the pipeline missed, the new test case, which gate gets it.

## Context Handoff

The output file from each phase is the handoff artifact. It carries forward what matters — the tool manages context naturally.

| Phase | Handoff Artifact |
|-------|-----------------|
| 1 → 2 | Problem statement |
| 2 → 3 | `requirements.md` |
| 3 → 4 | `architecture.md` |
| 4 → 5 | `threat_model.md` |
| 5 → 6 | Pipeline config + dummy product + `requirements.md` + `threat_model.md` |
| 6 → 7 | `tasks.md` |
| 7 → 8 | Working code + test results |

If a decision matters, it goes in the output file. If it is not in the output file, it does not carry forward.

### Phase Re-entry

When implementation reveals upstream flaws: identify which phase owns the flaw, re-run that phase with the current output + the finding, propagate changes forward. Document what triggered re-entry and what changed.

## Dual-Model Review

For architecture, threat model, CI/CD gates, and security-critical components: recommend the user have the output reviewed by a different model architecture (different company, different training). Different architectures fail differently.

The review is a structured argument: Generator produces → Reviewer attacks → Generator defends or acknowledges → Navigator (user) rules → Generator incorporates rulings. Neither model should accept the other's position without arguing its case. The user is the final judge.

## Waiver Pattern

When a rule must be broken, document: what is being skipped, why (the real reason), risk accepted, mitigation, owner, and expiry. An undocumented exception is a hidden liability.

## Reasoning Pipeline

Select pipeline depth based on problem complexity:
- Light: `RCAR → ToT → PMR`
- Standard: `FPR → RCAR → AdR → ToT → PMR`
- Political: `FPR → SMR → AdR → ToT → PMR`

## Minimal Viable Track

For smaller projects, the minimum is:
1. Problem statement (2-3 sentences)
2. `requirements.md`
3. Architecture sketch
4. 1-2 CI/CD gates (unit tests + no hardcoded secrets)
5. Dummy product that passes every gate

If you skip something, state what risk is being accepted.
