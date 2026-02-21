# Phase 3 — Architecture & Design

**Input:** Phase 2 `requirements.md`

## System Overview

[1-2 paragraphs describing the system at a high level. What are the major components and how do they interact?]

## Component Diagram

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│   Client /  │────>│  API Gateway │────>│   Service    │
│   UI Layer  │     │  / Router    │     │   Layer      │
└─────────────┘     └──────────────┘     └──────┬───────┘
                                                │
                                    ┌───────────┼───────────┐
                                    │           │           │
                              ┌─────▼──┐  ┌────▼───┐  ┌───▼────┐
                              │ Comp A │  │ Comp B │  │ Comp C │
                              └────┬───┘  └────┬───┘  └────┬───┘
                                   │           │           │
                              ┌────▼───────────▼───────────▼────┐
                              │        Data / Storage Layer      │
                              └─────────────────────────────────┘
```

## Components

### [Component A]

**Responsibility:** [Single sentence]
**Interface:** [Input → Output]
**Dependencies:** [What it needs, how it's injected]
**Testing:** [How to test in isolation — what gets mocked]

### [Component B]

**Responsibility:** [Single sentence]
**Interface:** [Input → Output]
**Dependencies:** [What it needs, how it's injected]
**Testing:** [How to test in isolation — what gets mocked]

### [Component C]

**Responsibility:** [Single sentence]
**Interface:** [Input → Output]
**Dependencies:** [What it needs, how it's injected]
**Testing:** [How to test in isolation — what gets mocked]

## External Dependencies

| Dependency | Purpose | Mock Strategy |
|-----------|---------|---------------|
| [e.g., AWS S3] | [e.g., Recipe storage] | [e.g., moto @mock_aws] |
| [e.g., PostgreSQL] | [e.g., State persistence] | [e.g., testcontainers] |
| [e.g., External API] | [e.g., Data enrichment] | [e.g., responses library] |

## Design Principles

- Dependency injection over hardcoded dependencies
- No hidden global state
- Every component testable in isolation
- [Project-specific principle]

## Gate Questions

- [ ] Can every component be tested in isolation?
- [ ] Where are the external dependencies and how are they mocked in tests?
- [ ] Does the architecture reflect the problem domain or what was easy to build?

## Decisions & Rejected Alternatives

| Decision | Alternative Rejected | Reason |
|----------|---------------------|--------|
| [e.g., Separate Lambda per function] | [e.g., Monolithic Lambda] | [e.g., Blast radius isolation — one compromised function doesn't expose others] |
| [e.g., Dependency injection via constructor] | [e.g., Module-level globals] | [e.g., Testability — can't mock globals cleanly] |

---

*This file is the sole input to Phase 4. Everything not written here does not carry forward.*
