# Phase 6 — Task Breakdown

**Input:** Phase 5 pipeline config + dummy product

## Tasks

### Task 1: [Component name]
**Files:** [what gets created or modified]
**Dependencies:** None (foundational)
**Acceptance criteria:**
- [ ] [Specific behavior from requirements.md] — verified by [unit test name/description]
- [ ] [Security gate] passes — maps to [threat_model.md risk]
**Pipeline gates exercised:** [e.g., unit tests, SAST, secret scan]

---

### Task 2: [Component name]
**Files:** [what gets created or modified]
**Dependencies:** Task 1
**Acceptance criteria:**
- [ ] [Specific behavior from requirements.md] — verified by [integration test name/description]
- [ ] [Security gate] passes — maps to [threat_model.md risk]
**Pipeline gates exercised:** [e.g., unit tests, integration tests, SCA]

---

### Task 3: [Component name] [P — can run in parallel with Task 2]
**Files:** [what gets created or modified]
**Dependencies:** Task 1
**Acceptance criteria:**
- [ ] [Specific behavior from requirements.md] — verified by [unit test name/description]
- [ ] [Custom security gate] passes — maps to [threat_model.md risk]
**Pipeline gates exercised:** [e.g., unit tests, SAST, custom gate]

---

## Task Order

```
Task 1 (foundational)
├── Task 2
├── Task 3 [parallel]
└── Task 4
    └── Task 5 (integration / E2E)
```

## Gate Questions

- [ ] Does every task map to at least one pipeline gate?
- [ ] Are acceptance criteria tied to specific requirements from requirements.md?
- [ ] Are security gates mapped to specific risks from threat_model.md?
- [ ] Can foundational tasks be verified before dependent tasks begin?

---

*This file is the sole input to Phase 7. Everything not written here does not carry forward.*
