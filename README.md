# Security-First AI Dev Methodology

**45% of AI-generated code fails security tests.** ([Veracode 2025](https://www.veracode.com/resources/analyst-reports/2025-genai-code-security-report/) — 100+ LLMs, 80 tasks, 4 languages). AI-generated code creates [1.7x more issues](https://www.coderabbit.ai/blog/state-of-ai-vs-human-code-generation-report) than human-written code, and security [degrades with each iteration](https://arxiv.org/html/2506.11022v1). Meanwhile, fully autonomous agents like Devin [succeed on 3 out of 20 tasks](https://www.theregister.com/2025/01/23/ai_developer_devin_poor_reviews/), and Cursor's AI-built browser had an [88% job failure rate](https://www.theregister.com/2026/01/22/cursor_ai_wrote_a_browser/).

Most LLM development methodologies focus on spec-first workflows, phase gates, and TDD — which help but aren't enough. This methodology adds three things that are typically handled ad-hoc or not at all: threat modeling as a required phase, cross-model adversarial review, and structured conversation architecture that keeps phases from polluting each other.

The [OWASP Top 10 for Agentic Applications (2026)](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/) identifies agent goal hijacking, rogue agents, and cascading failures as top risks — and recommends the principle of **least agency** as the foundational defense. Here's how that works in practice: models are the engine, you are the navigator.

---

## What this adds

### 1. Security as a first-class phase

Threat modeling is Phase 4 — before any code is written. Not a checklist bolted on after implementation. Every trust boundary, IAM blast radius, IaC configuration, and supply chain dependency is examined with adversarial intent.

### 2. Conversation architecture

The output file from each phase is the handoff artifact to the next. Nothing else carries forward. Modern agentic tools manage context naturally — through compaction, file-based context, or session architecture. The methodology defines what matters, the tool handles the mechanics.

### 3. Cross-model adversarial review

A single model reviewing its own output is structurally unreliable. The methodology prescribes using a different model architecture (different company, different training) to review security-critical decisions. Different architectures fail differently — the genuine disagreements between them are where the real signal lives.

---

## The phases

| Phase | Output | Gate question |
|-------|--------|---------------|
| 1. Problem Definition | Problem statement | What breaks if this isn't built? Why is code the right solution? |
| 2. Requirements | `requirements.md` | Is every requirement testable? What is explicitly out of scope? |
| 3. Architecture | `architecture.md` | Can every component be tested in isolation? |
| 4. Threat Model | `threat_model.md` | What is the worst thing an adversary can do at each trust boundary? |
| 5. CI/CD + Dummy Product | Pipeline config | What does a passing pipeline actually prove? |
| 6. Task Breakdown | `tasks.md` | Which pipeline gate validates each task? |
| 7. Implementation | Working code | Does the full pipeline pass? |
| 8. Production Feedback | Live system + new tests | What failures did production surface that the pipeline missed? |

Phases 1-5 are load-bearing walls — sequential and non-negotiable. Phase 6 onwards is Agile.

---

## Quick start: Minimal Viable Track

Not every project needs all eight phases at full depth. This is the 20% that delivers 80% of the value:

| What | Minimum output |
|------|----------------|
| Problem statement | 2-3 sentences: what breaks without this, why code solves it |
| `requirements.md` | Bullet list of what the system does and does not do |
| Architecture sketch | Component diagram or written description of boundaries |
| 1-2 CI/CD gates | Unit tests pass, no hardcoded secrets |
| Dummy product | Minimal implementation that passes every gate |

If you skip something, know what risk you are accepting. Skipping without awareness is the only true failure.

---

## Installation

### As a Claude Code skill (recommended)

```bash
# Clone into your personal skills directory
git clone https://github.com/Nellur35/security-first-ai-dev-methodology.git \
  ~/.claude/skills/security-first-methodology
```

Or copy into a specific project:

```bash
# Copy into your project's skills directory
mkdir -p .claude/skills/methodology
cp security-first-ai-dev-methodology/.claude/skills/methodology/SKILL.md \
  .claude/skills/methodology/SKILL.md
```

### As a CLAUDE.md drop-in

Copy `CLAUDE-skill.md` to your project root alongside your existing `CLAUDE.md`. Claude Code will pick it up automatically.

### As a reference document

Read [`METHODOLOGY.md`](METHODOLOGY.md) — starts with a 15-minute Quick Start and full reference with worked rationale for every decision.

---

## Templates

The [`templates/`](templates/) folder contains output templates for each phase — the expected shape and depth at each gate. Copy a template, fill in the placeholders, answer the gate questions, and the completed file becomes the input to the next phase's conversation.

---

## Files in this repo

| File | Purpose |
|------|---------|
| [`METHODOLOGY.md`](METHODOLOGY.md) | Full reference document with rationale |
| [`CLAUDE-skill.md`](CLAUDE-skill.md) | Condensed skill file for project drop-in |
| [`.claude/skills/methodology/SKILL.md`](.claude/skills/methodology/SKILL.md) | Claude Code skill — full methodology |
| [`.claude/skills/intake/SKILL.md`](.claude/skills/intake/SKILL.md) | Claude Code skill — interactive Phase 1 intake |
| [`tools/`](tools/) | Standalone prompts that work in any AI model |
| [`templates/`](templates/) | Phase output templates showing expected shape and depth |

---

## Who this is for

- Developers building production systems with AI coding tools
- Teams that need security review integrated into their AI workflow, not bolted on
- Anyone who has watched an LLM produce code that "looks correct" and passes tests while building the wrong thing

## Who this is NOT for

- Quick scripts and throwaway projects (use the Minimal Viable Track or skip this entirely)
- Teams that want full automation with no human judgment (this methodology requires a human navigator)

---

## Background

This methodology was developed through hands-on AI-assisted development — trial, error, pushback, and verification against reality. The security-first approach emerged from observing that most LLM development workflows optimize for speed and structure without treating security as a load-bearing phase.

The core insights — security before code, isolated conversations per phase, adversarial cross-model review — are independently validated by [OWASP's Agentic AI research](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/), [Veracode's 2025 GenAI security report](https://www.veracode.com/resources/analyst-reports/2025-genai-code-security-report/), [CrowdStrike's multi-agent red team systems](https://www.crowdstrike.com/en-us/blog/secure-ai-generated-code-with-multiple-self-learning-ai-agents/), and what people are starting to call [context engineering](https://blog.langchain.com/context-engineering-for-agents/).

The core philosophy: models guess what you probably want to hear, and they're good enough at it to fool you. Your role is navigator and judge. Their role is engine.

---

## License

MIT

---

## Author

Asaf Yashayev
