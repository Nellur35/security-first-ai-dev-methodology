# Adversarial Review Findings: CloudJanitor Brain Agent

**Generator:** Claude Sonnet 4.5 (Phases 1-3)
**Reviewer:** Claude Opus 4.6 (adversarial mandate)
**Navigator:** Claude Opus 4.6 (judgment)
**Date:** 2026-02-21

This document records the findings from the dual-model review of Phases 1-3 and the Navigator's rulings on each.

---

## Findings Accepted (incorporated into corrected output)

### P0 — Deployment authorization undefined
**Finding:** The Generator specified "explicit customer approval" without defining the mechanism — no authentication, no replay protection, no audit trail for the highest-risk action in the system.
**Resolution:** Added REQ-F019 (signed, non-replayable approval tokens with 15-min expiry), REQ-F020 (access model), and redesigned the Deployment Orchestrator as a separate authenticated action with 6-point verification.

### P0 — Indirect prompt injection via environment data
**Finding:** assess_environment scans customer resources. Resource names, tags, and policy descriptions are attacker-controlled strings that get embedded directly into the Bedrock prompt. The 20-pattern blocklist only scans user input.
**Resolution:** Added Environment Assessor sanitization rules, prompt data/instruction separation with explicit delimiters, and Output Validator as a distinct component. Prompt injection defense is now 4 layers, not a blocklist.

### P0 — No input size limit
**Finding:** No maximum request size creates trivial cost-explosion and DoS vector.
**Resolution:** Added 2000-character limit in REQ-F001 and Request Parser component.

### P1 — Stateless contradicts multi-step flow
**Finding:** "Stateless" design but Definition of Done describes request → recommend → approve → deploy → verify — which requires state between steps.
**Resolution:** Resolved via signed approval tokens (client-managed state). The server remains stateless. The approval token carries the recommendation context. EX-007 updated to clarify.

### P1 — No output validation between LLM and deployment
**Finding:** LLM output was trusted directly. Recipe IDs could be hallucinated; costs could be wrong.
**Resolution:** Added Output Validator as component 8 — a hard boundary between LLM output and any action. All customer-facing data (cost, compliance) comes from the recipe catalog, not the LLM.

### P1 — No rate limiting or concurrency model
**Finding:** No protection against concurrent request floods or cost exhaustion.
**Resolution:** Added Auth Gate (component 1) with rate limits (10 concurrent, 100/hour), plus REQ-NF016 for aggregate daily/monthly cost caps.

### P1 — System prompt caching without defensive wrapper
**Finding:** System prompt in Lambda global scope as a plain string — any logging, serialization, or debug output leaks it.
**Resolution:** Added SecurePrompt wrapper class that raises exceptions on str(), repr(), pickle, and only exposes the value through a dedicated for_bedrock() method.

### P2 — No rollback strategy
**Finding:** System deploys CloudFormation but has no undo mechanism.
**Resolution:** Added REQ-NF015 — pre-deployment state capture, rollback on compliance decrease or failure.

### P2 — No versioning
**Finding:** Recipes, system prompt, and model version all unversioned.
**Resolution:** Added REQ-NF017 (recipe versioning) and REQ-NF018 (model version pinning with regression testing).

### P2 — Definition of Done was a single scenario
**Finding:** DoD described one happy path, not categorical criteria.
**Resolution:** Rewrote as 9 categorical acceptance criteria with the scenario as one example.

### P2 — 99.5% valid response rate undefined
**Finding:** "Valid response" had no definition.
**Resolution:** Defined as: conforms to JSON schema, all recipe IDs exist in catalog, reasoning trace present, latency within target.

---

## Findings Accepted with Reduced Weight

### "5 recipes don't need an LLM"
**Finding:** A decision tree with 5 options doesn't require a foundation model. The LLM is here for UX, not reasoning necessity.
**Ruling:** Technically true at current scale. But the value isn't just routing — it's synthesizing assessment data, ordering deployments, and generating Pre-Mortem warnings. Added an honest caveat in Phase 1 acknowledging this.

### "Zero internet egress provides less security than implied"
**Finding:** VPC endpoints are standard practice, not a differentiator. Data can still exfiltrate via any AWS service endpoint.
**Ruling:** Fair. VPC endpoints are a necessary control, not a sufficient one. The architecture doesn't oversell this — it's listed as one layer in a 12-layer defense model.

---

## Findings Rejected

### "Mock-based tests produce false confidence"
**Finding:** boto3 Stubber mocks return exactly what you define. Gap between mock and production is where failures live.
**Ruling:** True, but the methodology separates unit tests (mocked, Phase 7) from integration tests (real services, Phase 5 CI/CD design). The Generator was doing Phase 3 architecture, which designs for unit testability. Integration test requirements come in Phase 5. Added a note in Phase 3 acknowledging this.

### "Phase 4 hasn't been executed"
**Finding:** The Generator produced Phases 1-3 but didn't do Phase 4 (Threat Model).
**Ruling:** The Generator was asked for Phases 1-3. Phase 4 is the next phase. This is the methodology working as designed, not a violation.

---

## Spikes Recommended Before Phase 4

1. **Latency:** Measure Bedrock p95 for 3-stage Standard pipeline under load
2. **Complexity router accuracy:** Test deterministic classifier against 50 real-world variations
3. **Environment data injection:** Test sanitization approach against real prompt injection via resource names/tags
