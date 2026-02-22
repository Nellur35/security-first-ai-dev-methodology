---
name: gate-check
description: >
  Verify exit criteria for any methodology phase. Auto-detects
  current phase from existing artifacts in the project. Activates
  when the user asks to check gates, wants to verify readiness,
  or is about to move to the next phase.
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash
---

# Gate Check

Verify you've met the exit criteria before moving forward.

## Step 1: Detect Current Phase

Glob for artifacts to figure out where the user is:

| Artifacts Found | Likely Finishing |
|----------------|-----------------|
| Nothing | Phase 1 |
| `problem_statement.md` or `reconstruction_assessment.md` | Phase 1 -> entering Phase 2 |
| `requirements.md` | Phase 2 -> entering Phase 3 |
| `architecture.md` | Phase 3 -> entering Phase 4 |
| `threat_model.md` | Phase 4 -> entering Phase 5 |
| Pipeline config (.github/workflows/, Jenkinsfile, etc.) | Phase 5 -> entering Phase 6 |
| `tasks.md` | Phase 6 -> entering Phase 7 |
| Implementation code + tests | Phase 7 (per task) |
| Deployed system | Phase 8 |

If ambiguous, ask: "Which phase are you checking?"

## Step 2: Read the Artifact

Read the artifact for the detected phase.

## Step 3: Evaluate Gates

### Phase 1 — Problem
- [ ] What breaks in the real world if this is not built?
- [ ] Why is code the right solution and not a process change, config, or existing tool?

**Proves:** The project has a real-world justification.
**Doesn't catch:** Whether the problem statement is too broad or solves a symptom.

### Phase 2 — Requirements
- [ ] Is every requirement testable?
- [ ] What is explicitly out of scope?
- [ ] What does done look like in reality, not on a dashboard?

**Proves:** Requirements are concrete, testable, and bounded.
**Doesn't catch:** Missing requirements you haven't thought of. Use `/review` to surface gaps.

### Phase 3 — Architecture
- [ ] Can every component be tested in isolation?
- [ ] Where are external dependencies and how are they mocked?
- [ ] Does the architecture reflect the problem domain or what was easy to build?

**Proves:** The system is structurally testable and the design is intentional.
**Doesn't catch:** Whether the architecture handles adversarial conditions. That's Phase 4.

### Phase 3.5 — Discovery Spike (if applicable)
- [ ] Was the assumption validated or disproven?
- [ ] Is the spike code discarded?
- [ ] Is architecture.md updated with the finding?

**Proves:** Unverified assumptions were tested against reality.

### Phase 4 — Threat Model
- [ ] What is the worst thing an adversary can do at each trust boundary?
- [ ] If the IAM execution role is compromised, what is the blast radius?
- [ ] Does the IaC have the same threat coverage as the application code?

**Proves:** Every trust boundary has been examined under adversarial conditions.
**Doesn't catch:** Novel attack vectors or zero-days. Use penetration testing and production monitoring.

### Phase 5 — CI/CD Pipeline
- [ ] What does a passing pipeline actually prove?
- [ ] Which gate catches which failure mode?
- [ ] Does the dummy product exercise every component?

**Proves:** The pipeline is designed to verify requirements and mitigate threats.
**Doesn't catch:** Whether gates are correctly implemented (verified when the pipeline runs).

### Phase 6 — Tasks
- [ ] Does every task produce a component the pipeline can validate?
- [ ] Are acceptance criteria tied to specific pipeline gates?
- [ ] Is every task independently testable?
- [ ] Is "done" defined as passing every gate, not working locally?

**Proves:** Work is structured so the pipeline validates each increment.

### Phase 7 — Implementation (per task)
- [ ] All acceptance criteria from tasks.md checked off?
- [ ] Full pipeline passes (not just locally)?
- [ ] Zero new warnings or regressions?

**Proves:** The code meets its acceptance criteria and the pipeline confirms it.
**Doesn't catch:** Production-only failure modes, performance under real load.

### Phase 8 — Production
- [ ] Are production failures being captured and converted into new tests?
- [ ] Are new tests being added to the pipeline?
- [ ] Is the pipeline becoming more comprehensive over time?

**Proves:** The system learns from production.

## Step 4: Output

### Gate Check: Phase [N] — [Name]

| Gate Question | Status | Evidence / Gap |
|--------------|--------|---------------|
| [question] | PASS / FAIL | [what satisfies it or what's missing] |

**If all pass:** "Phase [N] gates pass. Handoff artifact for Phase [N+1] is [artifact]. Consider `/review` for adversarial review before proceeding."

**If any fail:** "Phase [N] has [X] open gates. Address these before proceeding:" then list the specific gaps.

## Style

- Answer gates with specifics from the artifact, not "yes."
- If a gate can't be answered concretely, it's not met.
- A gate you can't answer is a gap you'll pay for later.
