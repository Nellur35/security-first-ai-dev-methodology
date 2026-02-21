# Phase 1 — Problem Definition

## Problem Statement

[2-3 sentences. What breaks in the real world without this? Why is code the right solution?]

**Example:**

> Users manually configure AWS security services across multiple accounts. This takes 2-4 hours per account, configurations drift within weeks, and misconfigurations are the #1 source of cloud security incidents. Code solves this because the configuration logic is deterministic and repeatable — a process change cannot enforce consistency across accounts over time.

## Gate Questions

- [ ] What breaks in the real world if this is not built?
- [ ] Why is code the right solution and not a process change, a configuration, or an existing tool?

## Alternatives Considered and Rejected

| Alternative | Why Rejected |
|-------------|-------------|
| [e.g., Manual process with checklist] | [e.g., Does not prevent configuration drift over time] |
| [e.g., Existing tool X] | [e.g., Costs $50K/year, requires dedicated team, overkill for this scale] |
| [e.g., Cloud-native service Y] | [e.g., Aggregates findings but does not deploy controls] |

---

*This file is the sole input to Phase 2. Everything not written here does not carry forward.*
