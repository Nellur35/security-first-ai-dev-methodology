# Worked Examples

## CloudJanitor — AI-Powered AWS Security Configuration Agent

This example shows Phases 1-5 of the methodology applied to a real project in development: an AI-powered AWS security configuration agent being built as an AMI product.

| Phase | File | What it demonstrates |
|-------|------|---------------------|
| 1. Problem | [`phase-1-problem.md`](cloudjanitor/phase-1-problem.md) | Gate questions answered explicitly. Alternatives to code rejected with reasoning. |
| 2. Requirements | [`phase-2-requirements.md`](cloudjanitor/phase-2-requirements.md) | Testable requirements, explicit exclusions, Decisions & Rejected Alternatives log. |
| 3. Architecture | [`phase-3-architecture.md`](cloudjanitor/phase-3-architecture.md) | Component boundaries, dependency injection for testability, interface definitions. |
| 4. Threat Model | [`phase-4-threat-model.md`](cloudjanitor/phase-4-threat-model.md) | **The key differentiator.** IAM blast radius analysis, prompt injection vectors, CFN templates as attack surface, error message information leakage — all caught before code was written. |
| 5. CI/CD | [`phase-5-cicd.md`](cloudjanitor/phase-5-cicd.md) | 6-job pipeline mapped to threat model, dummy product definition, two documented waivers. |

### Why this example matters

This project was initially built **without** the methodology. The result:
- 6 tests covering 230 functions
- A build-backend that didn't exist (`setuptools.backends._legacy:_Backend`)
- 11 CI failures on first push
- Error messages leaking AWS account IDs and IAM role ARNs

The Phase 4 threat model and Phase 5 gate questions would have caught every one of these issues before implementation began.
