---
name: security-first-methodology
description: >
  Orchestrator for the security-first development methodology.
  Routes to the right skill for each phase. Activates when the
  user starts a new project, asks about methodology, or needs
  to know what phase to work on next.
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash
---

# Security-First AI Dev Methodology

You are the engine. The user is the navigator and judge. Gate questions decide when a phase is complete. Phases 1-5 are sequential. Phase 6 onwards is iterative.

## Phase Routing

| Phase | What to Do | Skill |
|-------|-----------|-------|
| 1 — Problem | Define the real-world need | `/intake` |
| 2 — Requirements | Define what, not how. Use `templates/phase-2-requirements.md` | Direct work |
| 3 — Architecture | Design for testability. Offer `/review` when done | Direct work |
| 4 — Threat Model | Attack every trust boundary | `/threat-model` |
| 5 — CI/CD Pipeline | Pipeline defines done. Offer `/review` when done | Direct work |
| 6 — Tasks | Break into pipeline-validatable units | Direct work |
| 7 — Implementation | Code + tests. Use `/gate-check` per task | Direct work |
| 8 — Production | Feed failures back into the pipeline | Direct work |

## Available at Any Phase

| Need | Skill |
|------|-------|
| Check exit criteria before moving on | `/gate-check` |
| Adversarial review of any artifact | `/review` |
| Scan an existing codebase or CI/CD | `/audit` |

## Context Handoff

| Phase | Handoff Artifact |
|-------|-----------------|
| 1 -> 2 | `problem_statement.md` or `reconstruction_assessment.md` |
| 2 -> 3 | `requirements.md` |
| 3 -> 4 | `architecture.md` |
| 4 -> 5 | `threat_model.md` |
| 5 -> 6 | Pipeline config + dummy product + `requirements.md` + `threat_model.md` |
| 6 -> 7 | `tasks.md` |
| 7 -> 8 | Working code + test results |

If it's not in the output file, it doesn't carry forward.

## Phase Re-entry

Implementation will reveal upstream flaws. That's the process working. Identify which phase owns the flaw, re-run it with the current output + the finding, propagate changes forward.

## Two Unbreakable Rules

1. Tests verify behavior against requirements -- not execute lines of code.
2. Pipeline gates are never weakened to make things pass.

## Full Reference

See `METHODOLOGY.md` for rationale, worked examples, the reasoning pipeline, and the dual-model review system.
