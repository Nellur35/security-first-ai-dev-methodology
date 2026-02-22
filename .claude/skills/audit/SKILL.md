---
name: codebase-audit
description: >
  Scan an existing codebase and CI/CD pipeline. Maps architecture,
  test coverage, security controls, and pipeline gates. Outputs a
  gap analysis against the security-first methodology. Activates
  when the user asks to audit a codebase, wants to understand what
  exists, or needs to figure out where to start with an existing
  project.
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash
---

# Codebase & CI/CD Audit

Scan what exists. Map what's covered. Identify what's missing.

## Step 1: Scan CI/CD Configuration

Glob for pipeline configs:
- `.github/workflows/*.yml`, `.github/workflows/*.yaml`
- `Jenkinsfile`, `jenkins/*.groovy`
- `.gitlab-ci.yml`
- `.circleci/config.yml`
- `bitbucket-pipelines.yml`
- `azure-pipelines.yml`, `.azure-pipelines/*.yml`
- `.travis.yml`
- `Makefile` (look for test/lint/scan targets)
- `docker-compose*.yml` (look for test services)
- `Taskfile.yml`, `justfile`

Also check:
- `package.json` scripts section (test, lint, build, scan)
- `pyproject.toml` / `setup.cfg` tool sections
- `Dockerfile` (multi-stage builds with test stages)

Read each config. For each pipeline, extract:
- What triggers it (push, PR, schedule, manual)
- What jobs/stages run
- What tools it uses (test runners, linters, scanners)
- Whether it blocks merge or is advisory
- What it does NOT cover

## Step 2: Map Gates to Methodology

Check each methodology gate against what you found:

| Gate | What to Look For | Status |
|------|-----------------|--------|
| Unit tests | Test runner execution, coverage thresholds | Found / Missing / Partial |
| Integration tests | Multi-service test execution, docker-compose test targets | Found / Missing / Partial |
| E2E tests | Browser/API test execution | Found / Missing / Partial |
| SAST | semgrep, CodeQL, Bandit, SonarQube | Found / Missing / Partial |
| SCA / dependency scan | pip-audit, npm audit, Snyk, Dependabot, Renovate | Found / Missing / Partial |
| Secret scanning | gitleaks, truffleHog, git-secrets | Found / Missing / Partial |
| Container scanning | Trivy, Snyk container, Docker Scout | Found / Missing / Partial |
| IaC scanning | tfsec, checkov, cfn-lint, cfn-nag | Found / Missing / Partial |
| Linting | Language-specific linters (eslint, ruff, golangci-lint) | Found / Missing / Partial |
| Type checking | mypy, tsc, etc. | Found / Missing / Partial |

## Step 3: Map Codebase Architecture

Glob and read to understand the structure:
- Top-level directories and their purpose
- Primary language(s) and framework(s)
- Entry points (main files, route definitions, handlers)
- Configuration files and their scope
- Dependencies from lock files (package-lock.json, poetry.lock, go.sum, Cargo.lock)
- Database models or schema files
- README files in subdirectories

Output a component map: components, responsibilities, dependencies between them, external integrations, where sensitive data moves.

## Step 4: Scan Tests

Glob for test files:
- `tests/`, `test/`, `__tests__/`, `spec/`
- `*_test.go`, `*_test.py`, `*.test.js`, `*.test.ts`, `*.spec.js`, `*.spec.ts`
- `test_*.py`, `*Test.java`, `*_spec.rb`

Analyze:
- Test file count vs source file count
- Which components have tests, which don't
- Types of tests (unit, integration, E2E -- infer from file location and imports)
- Whether tests verify behavior or just execute lines (check for meaningful assertions)
- Coverage config (pytest --cov, jest --coverage, go test -cover)

## Step 5: Scan Security Artifacts

Glob for:
- `threat_model.md`, `threat-model.md`, `THREAT_MODEL.md`
- `security.md`, `SECURITY.md`, `.security/`
- `docs/security/`
- `.env`, `.env.example` (check for secrets patterns)
- IAM policy files, CloudFormation/Terraform with IAM
- Dockerfile USER directives (running as root?)
- Network policy files

## Step 6: Check Methodology Artifacts

Which phase outputs exist?
- `problem_statement.md` or `reconstruction_assessment.md` (Phase 1)
- `requirements.md` (Phase 2)
- `architecture.md` (Phase 3)
- `threat_model.md` (Phase 4)
- Pipeline config (Phase 5)
- `tasks.md` (Phase 6)

## Step 7: Output

### Audit Report: [Project Name]

#### Pipeline Coverage

| Gate | Status | Details | Recommendation |
|------|--------|---------|---------------|
| [gate] | Covered / Missing / Partial | [what exists] | [what to add] |

#### Architecture Map

[Component map from Step 3]

#### Test Coverage

| Component | Has Tests | Test Type | Notes |
|-----------|-----------|-----------|-------|
| [component] | Yes / No | Unit / Integration / E2E | [coverage if available] |

#### Security Posture

| Area | Status | Finding |
|------|--------|---------|
| Threat model | Exists / Missing | [details] |
| Secret management | [pattern found] | [details] |
| Dependency pinning | Pinned / Unpinned | [details] |
| Container security | [status] | [details] |
| IaC security | [status] | [details] |

#### Methodology Phase Coverage

| Phase | Status | Artifact |
|-------|--------|----------|
| 1 — Problem | Done / Missing | [file if exists] |
| 2 — Requirements | Done / Missing | [file if exists] |
| 3 — Architecture | Done / Missing | [file if exists] |
| 4 — Threat Model | Done / Missing | [file if exists] |
| 5 — CI/CD | Done / Missing | [file if exists] |
| 6 — Tasks | Done / Missing | [file if exists] |

#### Recommended Entry Point

Based on what exists, start at Phase [N] because [reason].

#### Priority Actions

Top 3-5 gaps to close, ranked by impact:
1. [Security gaps first -- missing threat model, no secret scanning, no SAST]
2. [Pipeline gaps -- missing tests, no coverage threshold, advisory-only gates]
3. [Documentation gaps -- missing architecture doc, no requirements]

## Style

- Report what exists factually. Don't judge quality -- that's what `/review` is for.
- Be specific about file paths where you found things.
- If you can't determine something without running the suite, say so.
- The audit is a snapshot, not a verdict. It tells you where to start.
