# Reasoning Pipeline for Complex Decisions

*When a model fills ambiguity with statistically plausible but wrong answers, structured reasoning forces it past the first plausible output.*

---

## Why This Exists

Models optimize for "plausible response" -- not "thorough response." They stop at the first reasonable answer. This is called **satisficing**. A single reasoning mode cannot cover the full problem space of a complex decision. The solution is to chain multiple reasoning frameworks into a pipeline where each stage analyzes the problem from a different angle, and the output of each stage informs the next.

## The Permission Slip Effect

This is the single most important property of the pipeline.

Models are sycophantic by default -- they tell you what you want to hear. Research on LLM sycophancy shows that models trained via RLHF are incentivized to be agreeable, sometimes at the cost of accuracy. Adding explicit permission to disagree (e.g., "find why this is wrong") increases rejection of flawed reasoning dramatically.

Structured stages like Pre-Mortem ("assume this failed -- why?") and Adversarial Reasoning ("what is each party secretly protecting?") give the model explicit structural permission to surface uncomfortable truths it would otherwise suppress.

In cross-model testing (3 problems at varied complexity, 4 pipeline variants, Sonnet 4.5 runs evaluated by Opus 4.6), insights like "the mandate itself is contradictory," "the VP ego is driving this decision," and "maybe this platform should not exist at all" appeared **only** in pipeline variants that included Adversarial or Pre-Mortem stages. The baseline ("think step by step") suppressed all of them.

The pipeline does not make the model smarter. It gives the model permission to say what it already knows.

---

## Available Frameworks

| Framework | Abbreviation | What It Does | Use When |
|-----------|-------------|-------------|----------|
| **First Principles** | FPR | Validates assumptions, checks if the framing is correct | The brief might be flawed; something feels off; ambiguous problem |
| **Chain of Thought** | CoT | Establishes facts, timeline, sequential logic | You need to understand what actually happened |
| **Root Cause Analysis** | RCAR / 5 Whys | Finds structural causes, not symptoms | Surface solutions keep failing; recurring problems |
| **Graph of Thoughts** | GoT | Maps systemic interconnections and feedback loops | Multiple interconnected elements; situation is stuck |
| **Stakeholder Mapping** | SMR | Maps power and interest for each player | Organizational politics; need to build coalitions |
| **Adversarial Reasoning** | AdR | Models what each party is protecting and optimizing for | Conflict, resistance, hidden incentives |
| **Tree of Thoughts** | ToT | Generates and compares multiple strategic options with tradeoffs | Designing interventions; high-stakes decisions with multiple paths |
| **Pre-Mortem** | PMR | Assumes failure, works backward to identify why | Before committing to any major strategy |

### Research Basis

The individual frameworks have independent research backing:

- **Chain of Thought** -- Wei et al., Google Brain, 2022. Demonstrated significant improvements on reasoning tasks, varying widely by task type and model scale. Only effective at large scale (100B+ parameters). (arXiv:2201.11903)
- **Tree of Thoughts** -- Yao et al., Princeton, 2023. Outperforms CoT on tasks requiring deliberate planning and search. (arXiv:2305.10601)
- **Graph of Thoughts** -- Besta et al., ETH Zurich, 2023. Outperforms ToT on tasks requiring decomposition and recombination of partial results. (arXiv:2308.09687)
- **Pre-Mortem** -- Gary Klein, 2007 (HBR). Based on prospective hindsight research by Mitchell, Russo & Pennington, 1989, which found that imagining an event has already occurred increases ability to identify reasons for outcomes by 30%.
- **Root Cause Analysis / 5 Whys** -- Toyota Production System, 1950s. Established method for distinguishing symptoms from structural causes.
- **Stakeholder Mapping** -- Mendelow, 1981 (ICIS proceedings). Power-interest grid for organizational analysis.

**What is novel here** is the integration of these frameworks into sequenced pipelines, the selection logic for choosing which to apply, and the permission slip finding. The pipeline architecture is practitioner-tested, not peer-reviewed.

---

## Pipeline Variants

### Light Pipeline (3 stages)

For moderate-complexity decisions where the framing is clear but you need structured analysis.

```
RCAR -> ToT -> PMR
```

What this gives you: Root cause identification, structured option comparison with tradeoffs, and specific failure modes to design against.

### Standard Pipeline (5 stages) -- First Principles opener

For complex decisions, especially those with ambiguity, competing stakeholders, or where the brief itself might be flawed.

```
FPR -> RCAR -> AdR -> ToT -> PMR
```

What this gives you: Everything in Light, plus assumption validation at the start and stakeholder incentive mapping before options are generated.

### Standard Pipeline (5 stages) -- CoT opener

For complex decisions where the facts need to be established before analysis. Use this when you know the framing is sound but the situation is complicated.

```
CoT -> RCAR -> AdR -> ToT -> PMR
```

### Political / Organizational Pipeline (5 stages)

For decisions dominated by organizational dynamics and stakeholder management.

```
FPR -> SMR -> AdR -> ToT -> PMR
```

### Systems Pipeline (5 stages)

For problems with feedback loops, interconnected components, and emergent behavior.

```
FPR -> RCAR -> GoT -> ToT -> PMR
```

---

## When to Use Which

```
Simple, well-defined problem    -> No pipeline. Direct prompt.
Moderate, familiar problem      -> Light (3 stages)
Complex, multi-angle            -> Standard (5 stages)
High-stakes, deeply political   -> Political or full custom (5-7 stages)
```

### Selection Logic

Start with these questions:

1. **Is the brief itself potentially flawed?** Start with First Principles.
2. **Do you need to establish what happened?** Add Chain of Thought.
3. **Are surface solutions failing?** Add Root Cause (5 Whys).
4. **Multiple interconnected elements?** Add Graph of Thoughts.
5. **Organizational politics involved?** Add Stakeholder Mapping + Adversarial.
6. **Hidden incentives or conflict?** Add Adversarial Reasoning.
7. **Multiple possible approaches?** Add Tree of Thoughts.
8. **High stakes?** Add Pre-Mortem. (Always recommended.)

---

## First Principles vs Chain of Thought as Opener

Testing showed that **First Principles is the stronger opener for ambiguous or politically complex problems** -- it catches flawed premises before you invest in detailed analysis.

However, this is not universal. CoT as opener occasionally generates unique tactical solutions that FPR misses, because establishing the full fact pattern sometimes reveals options that assumption-checking does not.

**Rule of thumb:** If the problem statement might be wrong, start with FPR. If the problem statement is solid but the situation is complex, start with CoT.

---

## How to Apply

Two modes:

**Mode 1 -- In your own thinking.** Run the pipeline yourself before prompting. Frame the problem clearly, then give the model a well-structured question instead of a raw one.

**Mode 2 -- In the prompt itself.** Ask the model to work through each stage explicitly:

```
Walk me through this problem in stages:

1. FIRST PRINCIPLES: What assumptions am I making? Are they valid?
2. ROOT CAUSE (5 Whys): What is actually causing this?
3. ADVERSARIAL: What is each party protecting? What would shift them?
4. OPTIONS (Tree of Thoughts): Generate 3-4 approaches, evaluate tradeoffs, recommend one.
5. PRE-MORTEM: Assume this failed in 6 months. Why? What should I design against?

Problem: [describe your situation with as much context as possible]
```

Mode 2 is more expensive (3-5x more tokens) but produces richer output. Use it for high-stakes decisions. Use Mode 1 for everything else.

### The Intake Pattern

For complex or unfamiliar problems, use a meta-prompt before running the pipeline:

```
I need to [brief description of challenge].

Before I ask you to analyze this, generate an intake questionnaire for me.
What do you need to know about the people, organizational context,
history, constraints, and success criteria?

Ask me the questions, I will answer, then we will proceed with analysis.
```

This surfaces blind spots in your own briefing. The pipeline is only as good as the context you feed it.

---

## What the Pipeline Produces That Baseline Misses

Based on cross-model testing (Sonnet 4.5 generation, Opus 4.6 evaluation) across problems at three complexity levels:

| What | Baseline ("think step by step") | Pipeline |
|------|--------------------------------|----------|
| Questions the framing | Rarely | Consistently (FPR stage) |
| Structured option comparison | Single recommendation | 3-5 options with tradeoffs and probability estimates |
| Stakeholder incentives mapped | No | Yes (AdR stage) |
| Specific failure modes | Generic warnings or none | Named failure modes with concrete mitigations |
| Uncomfortable truths surfaced | Suppressed by default agreeableness | Surfaced via permission slip stages |

### Where pipeline value is highest

The value scales with problem complexity:

- **Simple, well-defined problems:** Pipeline produces the same answer as baseline but costs 3x more. Not worth it.
- **Medium problems:** Pipeline adds structured options and failure analysis. The jump from baseline to Light (3-stage) is the biggest value gain.
- **Complex, multi-stakeholder problems:** Pipeline is transformative. Adversarial and Pre-Mortem stages surface dynamics that baseline suppresses entirely.

### Where pipeline value is lowest

- Simple factual questions
- Low-stakes routine work
- Problems where the path forward is already clear
- Time-sensitive situations where speed matters more than depth

---

## Limitations

- Pipeline costs 3-5x more tokens and time than simple prompts
- The testing behind these findings is practitioner-level (cross-model, varied complexity) but not peer-reviewed
- AI outputs are working notes, not ground truth -- verify critical points independently
- Works best with rich context; thin briefings produce thin analysis regardless of pipeline
- "More thorough analysis" does not automatically mean "better decisions" -- that depends on what you do with the analysis
- The specific framework sequences have not been validated as optimal across all domains

---

*The pipeline does not make the model smarter. It makes the model honest.*
