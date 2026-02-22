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

The user is the navigator and judge. You are the engine. Gate questions decide when a phase is complete. Phases 1-5 are sequential and non-negotiable. Phase 6 onwards is iterative.

## Standalone Tools

The `tools/` directory contains standalone prompts that work in any AI model. Use them directly or point users to them for focused work outside the full workflow.

| Tool | Purpose |
|------|---------|
| `tools/threat-model.md` | Generate a threat model from an architecture doc |
| `tools/review.md` | Adversarial review of any artifact |
| `tools/gate-check.md` | Gate questions for all 8 phases as a checklist |
| `tools/intake.md` | Interactive problem definition questionnaire |

## Phase Sequence

### Phase 1 — Problem
Answer: What is the actual problem? Why is code the right solution?
**Gate:** What breaks if not built? Why not a process change or existing tool?
**Output:** Problem statement (2-3 sentences) -> Phase 2.

### Phase 2 — Requirements
Define what the product must do, not how.
**Gate:** Every requirement testable? What is out of scope? What does done look like in reality?
**Output:** `requirements.md` with Decisions & Rejected Alternatives -> Phase 3.

### Phase 3 — Architecture
Design for testability: clean boundaries, dependency injection, no global state.
**Gate:** Every component testable in isolation? External dependencies mocked? Design reflects problem domain?
**Output:** `architecture.md` with components, interfaces, Decisions & Rejected Alternatives -> Phase 4. Offer adversarial review.

### Phase 4 — Threat Model
Attack every trust boundary. Use `tools/threat-model.md` for the full area table and output format.
**Gate:** Worst case at each boundary? IAM blast radius? IaC coverage matches app coverage?
**Output:** `threat_model.md` with risks, impact ratings, mitigations -> Phase 5. Offer adversarial review.

### Phase 5 — CI/CD Pipeline
The pipeline defines done. Select security gates from the architecture and threat model. Two unbreakable rules: tests verify behavior against requirements; gates are never weakened to pass.
**Gate:** What does passing prove? Which gate catches which failure? Dummy product exercises every component?
**Output:** Pipeline config + dummy product + gate definitions -> Phase 6. Offer adversarial review.

### Phase 6 — Tasks
Each task: pipeline-validatable, acceptance criteria tied to gates, independently testable, done only when every gate passes.
**Output:** `tasks.md` -> Phase 7.

### Phase 7 — Implementation
Tests alongside code. Full pipeline must pass before moving to next task. Fix code, not gates.

### Phase 8 — Production Feedback
Deploy, monitor, convert failures into new pipeline tests.

## Context Handoff

| Phase | Handoff Artifact |
|-------|-----------------|
| 1 -> 2 | Problem statement |
| 2 -> 3 | `requirements.md` |
| 3 -> 4 | `architecture.md` |
| 4 -> 5 | `threat_model.md` |
| 5 -> 6 | Pipeline config + dummy product + `requirements.md` + `threat_model.md` |
| 6 -> 7 | `tasks.md` |
| 7 -> 8 | Working code + test results |

If a decision matters, it goes in the output file. If not in the output file, it does not carry forward.

## Phase Re-entry

When implementation reveals upstream flaws: identify which phase owns the flaw, re-run that phase with the current output + the finding, propagate changes forward. Document what triggered re-entry and what changed.

## Dual-Model Review

For architecture, threat model, CI/CD gates, and security-critical components: recommend review by a different model architecture. The review is a structured argument: Generator produces, Reviewer attacks, Generator defends, Navigator rules. See `tools/review.md` for the full process and output format.

## Reasoning Pipeline

Select depth based on problem complexity:
- **Light** (clear framing): `RCAR -> ToT -> PMR`
- **Standard** (ambiguous): `FPR -> RCAR -> AdR -> ToT -> PMR`
- **Political** (organizational): `FPR -> SMR -> AdR -> ToT -> PMR`

Pre-Mortem and Adversarial stages bypass default agreeableness. Use them when surfacing uncomfortable truths matters. See `reasoning-pipeline.md` for full reference.

## Waiver Pattern

When a rule must be broken, document: what is being skipped, why, risk accepted, mitigation, owner, expiry. An undocumented exception is a hidden liability.

## Minimal Viable Track

Minimum for any project: problem statement, `requirements.md`, architecture sketch, 1-2 CI/CD gates (unit tests + no hardcoded secrets), dummy product. If you skip something, state what risk is being accepted.
