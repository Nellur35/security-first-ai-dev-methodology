# Phase Output Templates

These templates show the expected shape and depth for each phase's output file. They are not a real project — they are a reference for what "done" looks like at each gate.

| Phase | Template | What to look for |
|-------|----------|-----------------|
| 1. Problem | [`phase-1-problem.md`](phase-1-problem.md) | Gate questions answered explicitly. Alternatives rejected with reasoning. |
| 2. Requirements | [`phase-2-requirements.md`](phase-2-requirements.md) | Every requirement testable. Explicit exclusions. Decisions & Rejected Alternatives log. |
| 3. Architecture | [`phase-3-architecture.md`](phase-3-architecture.md) | Component boundaries, mock strategies, dependency injection, testability. |
| 4. Threat Model | [`phase-4-threat-model.md`](phase-4-threat-model.md) | Trust boundaries mapped. IAM blast radius analyzed. Error handling reviewed. Supply chain examined. |
| 5. CI/CD | [`phase-5-cicd.md`](phase-5-cicd.md) | Every gate mapped to a threat or failure mode. Dummy product defined. Waivers documented. |

## How to use these

1. Copy the template for the phase you're working on
2. Replace the bracketed placeholders with your project's specifics
3. Answer every gate question before moving to the next phase
4. The completed file becomes the sole input to the next phase's conversation

## The key principle

Each file ends with the same reminder: *"This file is the sole input to the next phase. Everything not written here does not carry forward."*

If a decision matters, it goes in the file. If it's not in the file, it doesn't exist in the next conversation.
