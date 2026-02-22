# Tools

Standalone prompts for security-first development. Each tool works in any AI model -- Claude, Gemini, Cursor, Kiro, ChatGPT, or anything else.

## How to Use

1. Open any AI chat.
2. Paste the contents of the tool file as the prompt.
3. Paste your input (architecture doc, threat model, phase artifacts, etc.) where indicated.
4. Get structured output.

No methodology knowledge required. Each tool is self-contained.

## Available Tools

### `threat-model.md`
Generate a structured threat model from an architecture document. Examines trust boundaries, IAM blast radius, secrets lifecycle, data lifecycle, supply chain (including LLM-generated code), and 10 other areas. Outputs a complete `threat_model.md`.

**Input:** `architecture.md` or system description
**Output:** `threat_model.md` with risks, impact ratings, and mitigations

### `review.md`
Adversarial review of any development artifact. Finds what is wrong, not whether it is good. Includes review lenses for architecture, threat models, and CI/CD pipelines. Produces structured findings with severity ratings and a disagreement table for navigator judgment.

**Input:** Any artifact (architecture, threat model, pipeline config, design doc)
**Output:** Structured findings with severity, impact, and recommended actions

### `gate-check.md`
Quick-reference checklist of exit criteria for all 8 phases. Each gate lists what it proves and what it does not catch. Use it to verify you have met the bar before moving to the next phase.

**Input:** The phase number you are checking
**Output:** Pass/fail for each gate question, with gaps identified

### `intake.md`
Interactive questionnaire for Phase 1 problem definition. Asks one question at a time, pushes back on vague answers, redirects solution-jumping back to the problem. Supports both greenfield projects and existing codebases.

**Input:** Describe your project
**Output:** `problem_statement.md` (greenfield) or `reconstruction_assessment.md` (existing project)

## Relationship to the Full Methodology

These tools extract focused capabilities from the [Security-First AI Dev Methodology](../METHODOLOGY.md). They are the executables; the methodology is the operating manual. You can use any tool independently, or use them together as part of the full 8-phase workflow.
