# Security-First AI Dev Methodology

**45% of AI-generated code fails security tests.** ([Veracode 2025](https://www.veracode.com/resources/analyst-reports/2025-genai-code-security-report/) — 100+ LLMs, 80 tasks, 4 languages). AI-generated code creates [1.7x more issues](https://www.coderabbit.ai/blog/state-of-ai-vs-human-code-generation-report) than human-written code, and security [degrades with each iteration](https://arxiv.org/html/2506.11022v1). Meanwhile, fully autonomous agents like Devin [succeed on 3 out of 20 tasks](https://www.theregister.com/2025/01/23/ai_developer_devin_poor_reviews/), and Cursor's AI-built browser had an [88% job failure rate](https://www.theregister.com/2026/01/22/cursor_ai_wrote_a_browser/).

Every existing LLM development methodology — Spec-Kit, BMAD, Superpowers, SPARC — ignores this. They handle spec-first workflows, phase gates, and TDD. None of them include threat modeling, adversarial cross-model review, or structured conversation architecture.

This one does.

The [OWASP Top 10 for Agentic Applications (2026)](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/) identifies agent goal hijacking, rogue agents, and cascading failures as top risks — and recommends the principle of **least agency** as the foundational defense. This methodology operationalizes that principle: models are the engine, you are the navigator.

---

## What makes this different

| Feature | Spec-Kit | Superpowers | BMAD | **This** |
|---------|----------|-------------|------|----------|
| Phase-gated workflow | Yes | Yes | Yes | **Yes** |
| Security / Threat modeling | No | No | No | **Phase 4** |
| Conversation architecture | No | No | No | **Yes** |
| Cross-model adversarial review | No | No | No | **Yes** |
| Test quality (beyond coverage) | Partial | Partial | Partial | **Yes** |

### 1. Security as a first-class phase

Threat modeling is Phase 4 — before any code is written. Not a checklist bolted on after implementation. Every trust boundary, IAM blast radius, IaC configuration, and supply chain dependency is examined with adversarial intent.

### 2. Conversation architecture

Each phase gets its own conversation. The output file from each phase is the context handoff to the next. Nothing else carries over. This is not a workaround for context window limits — it is a design principle that prevents context drift, contradictions, and the slow degradation that happens when a model tries to hold an entire project in one session.

### 3. Cross-model adversarial review

A single model reviewing its own output is structurally unreliable. The methodology prescribes using a different model architecture (e.g., Claude + Gemini) to review security-critical decisions. Different architectures fail differently — the genuine disagreements between them are where the real signal lives.

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

Read [`METHODOLOGY.md`](METHODOLOGY.md) — the full reference with worked rationale for every decision.

---

## Example output

The [`examples/template/`](examples/template/) folder shows what each phase's output looks like — a generic template you can adapt to any project. It demonstrates the shape and depth expected at each gate, without being tied to a specific domain.

---

## Files in this repo

| File | Purpose |
|------|---------|
| [`METHODOLOGY.md`](METHODOLOGY.md) | Full reference document with rationale |
| [`CLAUDE-skill.md`](CLAUDE-skill.md) | Condensed skill file for project drop-in |
| [`.claude/skills/methodology/SKILL.md`](.claude/skills/methodology/SKILL.md) | Claude Code skills ecosystem format |
| [`examples/template/`](examples/template/) | Phase output templates showing expected shape and depth |

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

This methodology was developed through hands-on AI-assisted development — trial, error, pushback, and verification against reality. The security-first approach emerged from observing that every LLM development methodology in the ecosystem optimizes for speed and structure while ignoring that the code being generated is statistically likely to contain vulnerabilities.

The core insights — security before code, isolated conversations per phase, adversarial cross-model review — are independently validated by [OWASP's Agentic AI research](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/), [Veracode's 2025 GenAI security report](https://www.veracode.com/resources/analyst-reports/2025-genai-code-security-report/), [CrowdStrike's multi-agent red team systems](https://www.crowdstrike.com/en-us/blog/secure-ai-generated-code-with-multiple-self-learning-ai-agents/), and the emerging discipline of [context engineering](https://blog.langchain.com/context-engineering-for-agents/).

The core philosophy: models are statistical machines. They are geniuses that need to be led by the hand. Your role is navigator and judge. Their role is engine.

---

## License

MIT

---

## Author

Asaf Yashayev
