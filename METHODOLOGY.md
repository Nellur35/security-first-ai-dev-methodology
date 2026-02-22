# LLM-Assisted Development

*A Structured Methodology for Building Real Products*

**Asaf Yashayev**

---

## Quick Reference

| Phase | What You Do | Output | Gate Question |
|-------|------------|--------|---------------|
| 1. Problem | Define the real-world need | Problem statement (2-3 sentences) | What breaks if this isn't built? Why is code the right solution? |
| 2. Requirements | Define what the system does and doesn't do | `requirements.md` | Is every requirement testable? What is out of scope? |
| 3. Architecture | Design components, boundaries, interfaces | `architecture.md` | Can every component be tested in isolation? |
| 3.5 Spike | Test unverified assumptions (optional) | Updated `architecture.md` | Was the assumption validated or disproven? |
| 4. Threat Model | Attack every trust boundary | `threat_model.md` | What is the worst an adversary can do here? |
| 5. CI/CD | Build the pipeline that defines "done" | Pipeline config + dummy product | What does a passing pipeline actually prove? |
| 6. Tasks | Break work into pipeline-validatable units | `tasks.md` | Which gate validates each task? |
| 7. Implementation | Write code — tests alongside, not after | Working code | Does the full pipeline pass? |
| 8. Production | Deploy, monitor, feed failures back | Live system + new tests | What did production catch that the pipeline missed? |

Phases 1-5 are sequential and non-negotiable. Phase 6 onwards is Agile. Each phase gets its own conversation — the output file is the only context that carries forward.

---

## 15-Minute Quick Start

Try the methodology on a trivial project before reading the full document. This proves it works before asking for your time.

**Project:** A URL shortener API.

**Phase 1 — Problem (2 min):**
> "What problem does a URL shortener solve? What breaks without it? Why is code the right solution?"

**Phase 2 — Requirements (3 min):**
> "Generate requirements.md for a URL shortener API. Include: functional requirements, non-functional requirements, explicit exclusions, definition of done. Add a Decisions & Rejected Alternatives section."

**Phase 3 — Architecture (3 min):**
> "Based on this requirements.md, design the architecture. Component diagram, interfaces, dependency injection boundaries. Every component must be testable in isolation. Include Decisions & Rejected Alternatives."

**Phase 4 — Threat Model (3 min):**
> "Based on this architecture.md, threat model every trust boundary. What can an adversary do? What is the blast radius? How do you mitigate?"

**Phase 5 — CI/CD (4 min):**
> "Based on this architecture.md and threat_model.md, generate a CI/CD pipeline config. Map each security gate to a specific threat. Build a dummy product that passes every gate."

**Result:** You now have a problem statement, requirements, architecture, threat model, pipeline, and dummy product — all in 15 minutes. Now read the full methodology below to understand the guardrails that prevent the 10 ways this usually goes wrong.

---

## The Core Philosophy

Models guess what you probably want to hear. They're good at it — good enough to fool you. Left alone, they'll produce code that looks correct, tests that pass, and pipelines that appear green — while building the wrong thing correctly. That's what this methodology is for.

Your role is navigator and judge. Not coder.

---

## On Methodology, Speed, and the Waterfall Objection

The first objection most people raise: this looks like Waterfall. Sequential phases, artifacts before code, gates before implementation. Everything the Agile movement spent twenty years arguing against.

The objection is wrong, but not for the reason you might expect.

### Foundation First, Sprints After

Phases 1 through 5 are load-bearing walls. You do not sprint your way into a foundation — you pour it once and pour it right. A threat model you iterate on is Swiss cheese. An architecture you discover through sprints is technical debt from day one.

Phase 6 onwards is Agile. Task breakdown is sprint planning. Implementation is sprint execution. The production feedback loop is your retrospective feeding the next cycle. The methodology is sequential where sequencing is load-bearing, and iterative everywhere else. Experienced engineers already do this. They pick the right model for the layer they are working on.

### On Speed

The second objection: this is too slow for modern development.

This confuses the methodology with a human team executing it manually. Two models running in parallel — one generating, one reviewing — compress the foundation phases dramatically. Architecture design with a simultaneous adversarial reviewer takes hours, not weeks. Threat modeling with dual-model review produces a more rigorous output in an afternoon than a human security review that takes a week to schedule.

The real question is whether skipping the foundation is faster in total. It never is. Time saved by skipping architecture is spent debugging. Time saved by skipping threat modeling is spent in incident response.

---

## Phase 1 — Define the Problem

Before writing a single prompt about code, answer two questions:

- What is the actual problem?
- Why does code solve it better than another approach?

Everyone thinks they can skip this. Then the model optimizes beautifully for the wrong objective and you end up with a working product that doesn't solve the real problem.

**Gate questions — do not proceed until answered:**
- What breaks in the real world if this is not built?
- Why is code the right solution and not a process change, a configuration, or an existing tool?

**Output:** A clear problem statement. 2-3 sentences. Defines the real-world need, not the technical solution.

---

## Phase 2 — Product Requirements

Define what the product must do, not how it will do it. Requirements are the translation layer between reality and code. Every requirement that is missing or ambiguous becomes a bug later.

- **Functional requirements:** what the system does
- **Non-functional requirements:** performance, security, reliability
- **Explicit exclusions:** what the system deliberately does NOT do
- **Definition of done:** what does success look like in reality, not on a dashboard

**Critical:** If requirements are unclear, stop and resolve them. You'll pay for it later. You always do.

**Gate questions — do not proceed until answered:**
- Is every requirement testable? If you cannot write a test for it, it is not a requirement.
- What is explicitly out of scope?
- What does done look like in reality, not in a CI dashboard?

**Output:** `requirements.md` with a **Decisions & Rejected Alternatives** section.

---

## Phase 3 — Architecture & Design

Design the system structure before any code is written. The architecture must reflect the problem domain, not convenience.

### Design for Testability

If you can't test it in isolation, you can't know if it works.

- Clean component boundaries
- Dependency injection over hardcoded dependencies
- No hidden global state
- Clear interfaces between components

The CI/CD pressure in Phase 5 will expose bad design immediately. Better to fix it now.

**Gate questions — do not proceed until answered:**
- Can every component be tested in isolation?
- Where are the external dependencies and how are they mocked in tests?
- Does the architecture reflect the problem domain or what was easy to build?

**Output:** `architecture.md` with component diagram, interface definitions, and Decisions & Rejected Alternatives log.

### Decisions & Rejected Alternatives

Output files are not just a list of final rules — they must capture intent. At the bottom of `requirements.md` and `architecture.md`, maintain a log of alternatives you actively rejected and why.

Without this, the implementing model in Phase 7 faces micro-decisions without historical context and will confidently optimize away constraints it does not know exist.

```
Requirement: Token lifespan is exactly 15 minutes.
Rejected: 60 minutes. Spike showed 60 minutes allows token exfiltration before anomaly detection triggers.
```

One sentence per rejected alternative. Low cost. Directly closes the context amnesia gap introduced by the handoff principle.

### Phase 3.5 — The Discovery Spike (Optional but Recommended)

Architecture cannot always be mapped perfectly in the abstract. If you have unverified assumptions about an API, a cloud service constraint, or performance latency — do not guess. Run a Spike.

Prompt the model to write a quick throwaway script to test the assumption against reality. Measure it. Record the result. Update the architecture based on what you learned. Then discard the code.

**The Golden Rule:** Spike code is radioactive. It exists to generate knowledge, not components. Its only output is an updated `.md` file.

**The Limits of Mechanical Enforcement**

The Conversation Architecture kill switch isolates the model — it prevents throwaway code from entering the implementation phase context window. The CI/CD gates will reject dirty scripts that lack proper tests and structure.

Neither control isolates the human.

The working script still exists on your local file system. Under deadline pressure, the temptation to copy-paste a Spike that already does the thing is high. A determined developer can write a meaningless mock test to drag that script past the coverage gate.

Keeping Spike code out of production is a team norm, not a technical control. The pipeline cannot save you from yourself. The discipline of the human judge to delete the Spike after extracting the knowledge is the only enforcement mechanism that works.

Stating this explicitly is not a weakness. Pretending a technical control exists when it is actually a cultural norm is security theater. The Waiver Pattern already demands this honesty — an undocumented exception is a hidden liability. The Spike requires the same treatment.

---

## Phase 4 — Threat Modeling

Architecture assumes a cooperative world. Threat modeling injects reality back in. For every component and trust boundary, ask:

- What does an adversary see here?
- What can they manipulate?
- What is the worst possible outcome?
- How would this component be abused at scale?

Security controls designed at this stage cost a fraction of what they cost after implementation. Most vulnerabilities aren't code bugs. They're architecture decisions nobody attacked on paper first.

### What to Examine

| Area | Questions to Ask |
|------|-----------------|
| Trust Boundaries | Where does control pass between components? Who is trusted? |
| Data Flows | Where does sensitive data travel? Who can intercept it? |
| Authentication | How does the system know who it's talking to? |
| Authorization | How does the system decide what is allowed? |
| External Dependencies | What happens if a dependency is compromised or unavailable? |
| Error Handling | Do error messages leak sensitive information? |
| Infrastructure & Cloud Boundaries | Where does code interact with the cloud provider? Are execution roles, parameter stores, and KMS keys explicitly scoped or implicitly broad? |
| IAM Blast Radius | If this execution role is hijacked, what is the worst case? What does it have access to beyond what it needs? |
| IaC & Configuration | Are infrastructure definitions version controlled and scanned? Can a misconfigured SSM parameter or overly permissive security group bypass all application-level controls? |
| Runtime Security | What happens after deployment? Container escape, SSRF, memory corruption, side-channel attacks? |
| Secrets Lifecycle | How are secrets provisioned, rotated, and revoked? What is the blast radius if a secret leaks? |
| Data Lifecycle | Where does data live, move, and die? Is deletion real or soft? Who has access at each stage? |
| Supply Chain | Are dependencies, CI/CD actions, IaC modules, build plugins, and dev tooling pinned? Could the LLM itself introduce compromised code? |

**Cloud reality check:** In modern cloud environments the application code is often the least interesting target. Catastrophic failures happen outside the code — in misconfigured IAM roles, exposed parameter stores, or infrastructure that was never threat modeled. Treat the infrastructure with the same adversarial rigor as the application.

**Gate questions — do not proceed until answered:**
- What is the worst thing an adversary can do at each trust boundary?
- If the IAM execution role is compromised, what is the blast radius?
- Does the IaC have the same threat coverage as the application code?

**Output:** `threat_model.md` with identified risks, impact ratings, and mitigations.

---

## Phase 5 — CI/CD Pipeline Design

Design the pipeline before any implementation begins. The pipeline tells you whether what you built matches what you said you'd build.

The pipeline shape follows from the architecture and threat model. Don't use templates or defaults. Build it to match what you are actually building.

### Test Strategy

| Level | What it Tests | Tools |
|-------|--------------|-------|
| Unit Tests | Single function in isolation, all dependencies mocked | pytest, jest, go test |
| Integration Tests | Components working together with real dependencies | Docker containers, test DBs |
| E2E Tests | Full system flow as a real user or system | Playwright, Cypress, Postman |
| Dummy Product | A reference implementation that runs through ALL tests | Same stack as production |

### Security Gates

Do not pick tools from a generic list. The right security gates follow from your architecture (Phase 3) and threat model (Phase 4). Use this prompt to generate a project-specific pipeline:

```
Based on this project's architecture and threat model:

Architecture: [paste architecture.md or summarize: language, framework, infra, deployment target]
Threat Model: [paste threat_model.md or summarize: top risks and trust boundaries]

Generate CI/CD security gates in two categories:

STANDARD GATES (select tools appropriate for this stack):
- SAST, SCA, secret scanning, container scanning, IaC scanning
- Justify each tool choice for this specific stack

CUSTOM GATES (derived from the threat model):
- For each high-impact risk in the threat model, define a gate that catches it
- Examples: IAM policy scope validation, VPC egress rule checks, encryption-at-rest
  enforcement, least-privilege verification, parameter store access auditing
- These are project-specific — they come from YOUR threat model, not a checklist

For every gate:
- Map it to a specific threat or trust boundary
- Define what it proves and what it does NOT catch
```

The standard gates (SAST, SCA, secrets, containers, IaC) are common across projects. The custom gates are where the real security value lives — they enforce the specific mitigations your threat model identified. A pipeline without custom gates is a generic checklist that misses your actual attack surface.

### Quality Gates

- **Coverage threshold** — set based on risk profile, not vanity. Coverage is a side effect of good tests, never a goal.
- **Linting** — enforce style and catch obvious errors
- **Type checking** — where applicable
- **Complexity limits** — flag functions that are too long or too complex

### The Two Unbreakable Rules

**Rule 1:** Tests must verify behavior against requirements — not execute lines of code. A test that passes without catching a real failure mode is noise that erodes trust in the pipeline.

**Rule 2:** Pipeline gates must never be weakened to make things pass. If something fails, fix the code or reconsider the architecture. Lowering thresholds, adding exclusions, or skipping checks just hides the decision you're actually making.

### The Dummy Product

Build a minimal reference product that exercises every component and passes every gate. Think of it as the canary. If a new test breaks the dummy product, you've caught a real problem before it reaches production code.

**Gate questions — do not proceed until answered:**
- What does a passing pipeline actually prove?
- Which gate catches which failure mode?
- Does the dummy product exercise every component?

**Output:** Pipeline config files + dummy product + all gate definitions.

---

## Phase 6 — Task Breakdown

Only after Phases 1-5 are complete do you break work into implementation tasks. Each task must:

- Produce a component that the pipeline can validate
- Have clear acceptance criteria tied to pipeline gates
- Be small enough to be independently testable
- Be considered done only when it passes every gate — not when it works locally

### Task Format

```
### Task [ID]: [Component name]
Files: [what gets created or modified]
Dependencies: [which tasks must complete first, if any]
Acceptance criteria:
- [ ] [Specific behavior from requirements.md] — verified by [unit/integration/E2E test]
- [ ] [Security gate X] passes — maps to [threat_model.md risk Y]
Pipeline gates exercised: [list which gates validate this task]
```

Order tasks so that foundational components (shared interfaces, data models, core utilities) come first. Later tasks build on earlier ones. Mark tasks that can run in parallel.

**Output:** `tasks.md` with acceptance criteria for each task.

---

## Phase 7 — Implementation

Now the model writes code. Not before.

- Write tests alongside code, never after
- Commit only what passes the full pipeline
- Each task is verified by the pipeline before moving to the next
- If the pipeline fails, fix the code — do not adjust the gate

**Per-task verification before moving to the next task:**

1. All acceptance criteria from `tasks.md` checked off
2. Full pipeline passes — not locally, the full pipeline
3. No new warnings or regressions introduced

---

## Phase 8 — Production Feedback Loop

Passing the CI pipeline is not the end. It is the beginning of a feedback loop.

1. Deploy to a live environment
2. Monitor for failures the pipeline did not catch
3. Collect logs and error patterns
4. Feed logs back to the model to generate new test cases
5. Add new tests to the pipeline
6. The pipeline becomes progressively more comprehensive

### Production Finding Format

```
### Finding: [what happened]
Failure mode: [what the pipeline missed and why]
New test: [test case that would have caught this]
Pipeline gate: [which gate gets this new test]
```

Do this. The tests you write before production will never be as good as the tests derived from real failure modes.

---

## The Dual-Model Review System

A single model reviewing its own output is unreliable. Self-critique is structurally weak. Use adversarial collaboration instead.

### Structure

| Role | What | Mandate |
|------|------|---------|
| Generator | The model producing the phase output | Produce the output for each phase |
| Reviewer | A different model architecture (different company, different training) | Find holes in logic, security, completeness |
| Judge | You | Resolve genuine disagreements, make final calls |

### Why Different Architectures

Two models from the same family share correlated blind spots. Different architectures fail differently. Models from different companies will surface more genuine disagreements than two instances of the same model.

### The Review Loop

The review is a structured argument. Both models make their case. You decide.

```
1. Generator produces phase output
2. Reviewer attacks it (adversarial mandate: find why this is wrong)
3. Generator responds — defends its decisions or acknowledges the gap
4. Navigator (you) rules on each disagreement
5. Generator incorporates the rulings and produces corrected output
```

Neither model should accept the other's position without arguing its case first. The Generator should defend sound decisions against the Reviewer's criticism. The Reviewer should not back down when the Generator pushes back if the finding is valid. The genuine disagreements — where both models have reasonable arguments — are where your judgment as navigator matters most.

If both models agree, it is probably right. If both models disagree, you have a real decision to make. If one model caves immediately, the review was not adversarial enough.

### How to Use the Reviewer

- Give the reviewer an explicitly adversarial mandate: find why this is wrong, not whether it is good.
- Do not tell the reviewer what to look for. Let it find what the generator missed.
- Feed the reviewer's findings back to the generator. Let the generator argue back.
- Do not automatically side with either model. Both can be wrong.
- You are the final judge. Make the call, document the reasoning, move on.

**When to use dual-model:** Architecture decisions, threat model, CI/CD gate definitions, security-critical components, anything where a mistake is expensive to fix later.

**When not required:** Boilerplate, simple scripts, obvious tasks with clear acceptance criteria.

---

## Reasoning Pipeline for Complex Decisions

When facing ambiguous or high-stakes decisions at any phase, apply structured reasoning before prompting.

The full reasoning pipeline reference -- including framework descriptions, pipeline variants, selection logic, and testing findings -- is in [`reasoning-pipeline.md`](./reasoning-pipeline.md).

### Quick Reference

**Light pipeline** (moderate decisions, clear framing):
```
RCAR -> ToT -> PMR
```

**Standard pipeline** (complex decisions, ambiguous framing):
```
FPR -> RCAR -> AdR -> ToT -> PMR
```

**Political / organizational pipeline:**
```
FPR -> SMR -> AdR -> ToT -> PMR
```

### Key Finding: The Permission Slip Effect

Pre-Mortem and Adversarial stages are not just analytical tools -- they structurally bypass the model's default agreeableness. Models suppress uncomfortable truths unless explicitly given permission to surface them. Naming a stage "Pre-Mortem" or "Adversarial Reasoning" provides that permission. In testing, critical insights like flawed premises and stakeholder conflicts appeared **only** in pipeline variants that included these stages.

### When to Use

- **No pipeline:** Simple, well-defined problems with clear solutions.
- **Light (3 stages):** Moderate decisions where you need structured options and risk identification.
- **Standard (5 stages):** Complex decisions with ambiguity, competing stakeholders, or where the brief itself might be wrong.

For simple problems the pipeline produces the same answer at 3x the cost. For complex multi-stakeholder problems it is transformative.

---

## Context Handoff

The output file from each phase is the handoff artifact to the next phase. Modern agentic tools (Claude Code, Cursor, Kiro, and others) manage context naturally — through compaction, file-based context, or session architecture. The methodology does not prescribe how your tool manages sessions. It defines what carries forward.

### Handoff Artifacts

| Phase | Handoff Artifact |
|-------|-----------------|
| Problem → Requirements | Problem statement |
| Requirements → Architecture | `requirements.md` |
| Architecture → Threat Model | `architecture.md` |
| Threat Model → CI/CD | `threat_model.md` |
| CI/CD → Tasks | Pipeline config + dummy product + `requirements.md` + `threat_model.md` |
| Tasks → Implementation | `tasks.md` |
| Implementation → Production | Working code + test results |

Phase 6 takes multiple inputs because task acceptance criteria must trace back to requirements and threat model risks.

### Phase Re-entry

Implementation will reveal upstream flaws. This is not a failure — it is the process working.

When Phase 7 surfaces an architecture problem, a missing requirement, or a threat not modeled:

1. Identify which phase owns the flaw (e.g., architecture → Phase 3, missing requirement → Phase 2)
2. Re-run that phase with the current output file + the finding
3. Work through the phase gates again with the new information
4. Propagate changes forward through all downstream phases

Re-entry is not a waiver. It does not need justification. Document what triggered the re-entry and what changed — one sentence is enough.

### Cross-Model Review at Handoff Points

The handoff between Phases 3→4, 4→5, and 5→6 is the natural point for cross-model review. Before starting the next phase, paste the output file into a different model with an adversarial prompt. This takes 60 seconds and catches blind spots the generator cannot see.

Ready-to-use review prompts:

**After Phase 3 (Architecture):**
> Review this architecture with an adversarial mandate. Find what is wrong, not whether it is good. Check: testability of every component, hidden dependencies, missing interfaces, whether the architecture reflects the problem domain or what was easy to build. For each finding: state the issue, the impact, and what should change. `[paste architecture.md]`

**After Phase 4 (Threat Model):**
> Review this threat model with an adversarial mandate. Find what is missing, not whether it is complete. Check: every trust boundary in the architecture, IAM blast radius, IaC configuration, supply chain risks, error handling information leakage. What would an attacker target first? What is the worst case the model didn't consider? `[paste threat_model.md]`

**After Phase 5 (CI/CD):**
> Review this CI/CD pipeline with an adversarial mandate. For each gate: does it actually catch what it claims to catch? What failure modes slip through? Are the custom security gates sufficient for the threat model risks? Is the dummy product exercising every component or just the happy path? `[paste pipeline config]`

Feed the reviewer's findings back to the generator and let it defend or acknowledge. You are the navigator — you decide which findings to incorporate.

### Why Handoff Artifacts, Not Decision Logs

Context Anchoring — feeding a growing Decision Log into every prompt — sounds like a solution to context drift but becomes the problem it tries to solve. A log that grows with every phase eventually consumes the context window. Don't fight the context window. Use it. Handoff artifacts carry forward only what matters.

**Rule:** If a decision is important enough to carry forward, it belongs in the output file. If it is not in the output file, it does not carry forward.

---

## The Waiver Pattern

Silent violations destroy pipeline reliability. But deadlines exist. Constraints exist. The problem is never breaking a rule — it's breaking one without acknowledging it.

An undocumented exception is the only true violation.

### When to Use a Waiver

- Weakening a coverage threshold
- Skipping a security scan for a release
- Skipping threat modeling under time pressure
- Deploying without the dummy product passing all gates
- Using one model instead of dual-model review for a critical decision

### The Waiver Template

| Field | What to Write |
|-------|--------------|
| What is being skipped or weakened | Name the specific gate, phase, or rule being bypassed |
| Why | The actual reason, not the meeting-friendly version |
| Risk accepted | What failure mode is now more likely? What is the worst case? |
| Mitigation | What compensating control exists, if any? |
| Owner | Who made this decision and is accountable for the risk |
| Expiry | When will this waiver be reviewed or the rule reinstated? |

The waiver does not need to be formal. A PR comment, a line in a decision log, a message in a team channel. Written, visible, and attributed.

---

## Working with an Existing Codebase

This methodology was built for greenfield projects. Most real work involves existing code — technical debt, missing documentation, decisions nobody remembers making. Build the full picture before touching anything.

### The Advantage of Not Being a Developer

A developer looks at existing code and sees code. The instinct is to read it, understand it, modify it. Wrong starting point. Start with: what problem was this built to solve, and why was it built this way? The model can read the code. You ask the questions.

### The Reconstruction Process

1. Feed the model the codebase and ask it to explain what problem this solves — not what it does technically, but what real-world need it serves.
2. Ask the model to map the components and their dependencies. Build the architecture picture you would have designed in Phase 3.
3. Ask the model what decisions were clearly made deliberately versus what looks accidental or improvised. This surfaces the invisible constraints.
4. Run threat modeling on what exists, not on what you wish existed. The attack surface is the current system.
5. Audit the existing CI/CD — if there is one. What does it actually test? What does it miss? What gates are there and are they meaningful?
6. Only after you have this picture do you decide what to change and in what order.

### What to Hope For, What to Prepare For

| Situation | Approach |
|-----------|----------|
| Good documentation exists | Use it as the starting point, verify it against the actual code |
| No documentation | The model reconstructs it — slower but doable |
| Modular architecture | Work component by component, the methodology applies cleanly |
| Monolith | Map it first, identify seams where components could be separated, work within constraints |
| No CI/CD at all | Build it from scratch using the current codebase as the dummy product |

The feedback loop still applies. Production failures still generate new tests. The difference is you're inheriting someone else's decisions, not designing from a clean slate. Understand before you touch.

---

## Minimal Viable Track

Not every project needs all eight phases at full depth. Start here. This is the 20% that delivers 80% of the value.

| What | Why It Cannot Be Skipped | Minimum Acceptable Output |
|------|--------------------------|--------------------------|
| Problem Statement | Without this the model optimizes for the wrong thing | 2-3 sentences: what breaks without this, why code solves it |
| `requirements.md` | Without this "done" has no definition | Bullet list of what the system does and does not do |
| Architecture Sketch | Without this the code has no structure to test against | Component diagram or written description of main components and their boundaries |
| 1-2 CI/CD Gates | Without this "passing" means nothing | At minimum: unit tests pass, no hardcoded secrets |
| Dummy Product | Without this the gates are never verified end to end | Minimal implementation that passes every gate you defined |

### What is Optional in the Minimal Track

- Full threat model — do a 15-minute version: what is the worst thing that can happen here?
- Dual-model review — one model is fine to start
- Full reasoning pipeline — use your judgment for simple decisions
- Complete security gate suite — add gates as the project grows
- Production feedback loop — activate once the system is live

Once the minimal track works, layer in the full methodology phase by phase.

**The rule:** If you skip something, know what risk you are accepting.

---

*The goal is not a passing pipeline. The goal is a system that correctly serves reality. The pipeline is just how you check.*
