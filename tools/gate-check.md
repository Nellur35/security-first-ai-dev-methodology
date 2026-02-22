# Gate Check Tool

Verify you have met the exit criteria for any phase before moving forward.

**Input:** The phase number you are checking (or "all" for a full audit).
**Output:** Pass/fail status for each gate question, with gaps identified.

---

## Phase 1 — Define the Problem

- [ ] What breaks in the real world if this is not built?
- [ ] Why is code the right solution and not a process change, a configuration, or an existing tool?

**Proves:** The project has a real-world justification and code is the right approach.
**Does not catch:** Whether the problem statement is too broad, too narrow, or solving a symptom instead of a root cause.

**Required output:** Problem statement (2-3 sentences defining the real-world need).

## Phase 2 — Product Requirements

- [ ] Is every requirement testable? (If you cannot write a test for it, it is not a requirement.)
- [ ] What is explicitly out of scope?
- [ ] What does done look like in reality, not on a dashboard?

**Proves:** Requirements are concrete, testable, and bounded.
**Does not catch:** Missing requirements you have not thought of yet. Use adversarial review (see `tools/review.md`) to surface gaps.

**Required output:** `requirements.md` with Decisions & Rejected Alternatives section.

## Phase 3 — Architecture & Design

- [ ] Can every component be tested in isolation?
- [ ] Where are the external dependencies and how are they mocked in tests?
- [ ] Does the architecture reflect the problem domain or what was easy to build?

**Proves:** The system is structurally testable and the design is intentional.
**Does not catch:** Whether the architecture handles adversarial conditions. That is Phase 4.

**Required output:** `architecture.md` with component diagram, interfaces, and Decisions & Rejected Alternatives.

## Phase 3.5 — Discovery Spike (if applicable)

- [ ] Was the assumption validated or disproven?
- [ ] Is the spike code discarded (not carried into implementation)?
- [ ] Is `architecture.md` updated with the finding?

**Proves:** Unverified assumptions were tested against reality.
**Does not catch:** Whether the developer actually deleted the spike code. This is a team norm, not a technical control.

## Phase 4 — Threat Modeling

- [ ] What is the worst thing an adversary can do at each trust boundary?
- [ ] If the IAM execution role is compromised, what is the blast radius?
- [ ] Does the IaC have the same threat coverage as the application code?

**Proves:** Every trust boundary has been examined under adversarial conditions and mitigations exist.
**Does not catch:** Novel attack vectors, zero-days, or threats that require running the system to discover. Use penetration testing and production monitoring to cover those.

**Required output:** `threat_model.md` with risks, impact ratings, and mitigations.

## Phase 5 — CI/CD Pipeline Design

- [ ] What does a passing pipeline actually prove?
- [ ] Which gate catches which failure mode?
- [ ] Does the dummy product exercise every component?

**Proves:** The pipeline is intentionally designed to verify requirements and mitigate identified threats.
**Does not catch:** Whether the gates are correctly implemented (that is verified when the pipeline runs), or failure modes not in the threat model.

**Required output:** Pipeline config files + dummy product + all gate definitions.

## Phase 6 — Task Breakdown

- [ ] Does every task produce a component the pipeline can validate?
- [ ] Are acceptance criteria tied to specific pipeline gates?
- [ ] Is every task independently testable?
- [ ] Is "done" defined as passing every gate, not working locally?

**Proves:** Work is structured so the pipeline validates each increment.
**Does not catch:** Whether task ordering is optimal or whether tasks are sized correctly. Those emerge during implementation.

**Required output:** `tasks.md` with acceptance criteria mapped to pipeline gates.

## Phase 7 — Implementation

- [ ] Are all acceptance criteria from `tasks.md` checked off for this task?
- [ ] Does the full pipeline pass (not just locally)?
- [ ] Are there zero new warnings or regressions?

**Proves:** The code meets its acceptance criteria and the pipeline confirms it.
**Does not catch:** Production-only failure modes, performance under real load, edge cases not covered by tests.

## Phase 8 — Production Feedback Loop

- [ ] Are production failures being captured and converted into new tests?
- [ ] Are new tests being added to the pipeline?
- [ ] Is the pipeline becoming more comprehensive over time?

**Proves:** The system learns from production and the pipeline evolves.
**Does not catch:** Failures that have not happened yet. The feedback loop is reactive by design -- it closes gaps as they appear.

---

## How to Use This

1. Pick the phase you are about to leave.
2. Answer every gate question with specifics, not "yes."
3. If any answer is vague or missing, you are not ready to move forward.
4. For phases 3, 4, and 5: run an adversarial review (`tools/review.md`) before proceeding.

The gates are exit criteria, not a formality. A gate you cannot answer concretely is a gap you will pay for later.
