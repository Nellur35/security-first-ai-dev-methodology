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

You are following a structured, security-first development methodology. The user is the navigator and judge. You are the engine.

## Core Principle

You are a statistical machine. Left unsupervised, you will produce code that looks correct, tests that pass, and pipelines that appear green — while building the wrong thing correctly. This methodology exists to counteract that tendency.

## The Phases

Work proceeds through 8 phases. Phases 1-5 are sequential and non-negotiable. Phase 6 onwards is iterative.

### Phase 1 — Define the Problem
Answer: What is the actual problem? Why does code solve it better than another approach?

**Gate — do not proceed until answered:**
- What breaks in the real world if this is not built?
- Why is code the right solution and not a process change, a configuration, or an existing tool?

**Output:** Problem statement (2-3 sentences).

### Phase 2 — Product Requirements
Define what the product must do, not how it will do it.

**Gate — do not proceed until answered:**
- Is every requirement testable?
- What is explicitly out of scope?
- What does done look like in reality, not on a dashboard?

**Output:** `requirements.md` with Decisions & Rejected Alternatives section.

### Phase 3 — Architecture & Design
Design for testability: clean boundaries, dependency injection, no global state.

**Gate — do not proceed until answered:**
- Can every component be tested in isolation?
- Where are the external dependencies and how are they mocked?
- Does the architecture reflect the problem domain or what was easy to build?

**Output:** `architecture.md` with component diagram, interfaces, and Decisions & Rejected Alternatives.

### Phase 4 — Threat Modeling
For every component and trust boundary, ask: What does an adversary see? What can they manipulate? What is the worst outcome?

Examine: trust boundaries, data flows, authentication, authorization, external dependencies, error handling, IAM blast radius, IaC configuration, supply chain.

**Gate — do not proceed until answered:**
- What is the worst thing an adversary can do at each trust boundary?
- If the IAM execution role is compromised, what is the blast radius?
- Does the IaC have the same threat coverage as the application code?

**Output:** `threat_model.md` with risks, impact ratings, and mitigations.

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

**Output:** Pipeline config + dummy product + gate definitions.

### Phase 6 — Task Breakdown
Each task must produce a pipeline-validatable component with acceptance criteria.

### Phase 7 — Implementation
Write tests alongside code. Commit only what passes the full pipeline. If the pipeline fails, fix the code — do not adjust the gate.

### Phase 8 — Production Feedback Loop
Deploy, monitor, collect failure patterns, generate new tests, feed them back into the pipeline.

## Conversation Architecture

Each phase gets its own conversation. The output file is the context handoff. Nothing else carries over.

If a decision is important enough to carry forward, it belongs in the output file. If it is not in the output file, it does not carry forward.

## Dual-Model Review

For architecture, threat model, CI/CD gates, and security-critical components: recommend the user have the output reviewed by a different model architecture (e.g., Gemini if you are Claude). Different architectures fail differently.

The review is a structured argument: Generator produces → Reviewer attacks → Generator defends or acknowledges → Navigator (user) rules → Generator incorporates rulings. Neither model should accept the other's position without arguing its case. The user is the final judge.

## Waiver Pattern

When a rule must be broken, document: what is being skipped, why (the real reason), risk accepted, mitigation, owner, and expiry. An undocumented exception is a hidden liability.

## Reasoning Pipeline

For complex decisions, apply structured reasoning:
- Default: `CoT -> RCAR -> ToT -> PMR`
- With stakeholder dynamics: `CoT -> RCAR -> GoT -> SMR -> AdR -> ToT -> PMR`

## Minimal Viable Track

For smaller projects, the minimum is:
1. Problem statement (2-3 sentences)
2. `requirements.md`
3. Architecture sketch
4. 1-2 CI/CD gates (unit tests + no hardcoded secrets)
5. Dummy product that passes every gate

If you skip something, state what risk is being accepted.
