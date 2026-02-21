# Phase 4 — Threat Model

**Input:** Phase 3 `architecture.md`

## Threat Context

[1-2 paragraphs. What makes this system interesting to an adversary? What is the worst thing that can happen if this system is compromised?]

## Trust Boundaries

```
┌─────────────────────────────────────────────────┐
│                 TRUST BOUNDARY 1                │
│  ┌──────────┐          ┌──────────────────┐     │
│  │  Client  │─────────>│  API / Gateway   │     │
│  └──────────┘          └────────┬─────────┘     │
│                    TRUST BOUNDARY 2              │
│                        ┌────────▼─────────┐     │
│                        │  Service Layer   │     │
│                        └────────┬─────────┘     │
│                    TRUST BOUNDARY 3              │
│                        ┌────────▼─────────┐     │
│                        │  Data / Infra    │     │
│                        └──────────────────┘     │
└─────────────────────────────────────────────────┘
```

## Threat Analysis by Trust Boundary

### Trust Boundary 1: Client → API

| Threat | Impact | Likelihood | Mitigation |
|--------|--------|------------|------------|
| [e.g., Unauthorized access] | [High/Medium/Low] | [High/Medium/Low] | [e.g., Authentication required, rate limiting] |
| [e.g., Input injection] | [High/Medium/Low] | [High/Medium/Low] | [e.g., Input validation, parameterized queries] |

### Trust Boundary 2: API → Service

| Threat | Impact | Likelihood | Mitigation |
|--------|--------|------------|------------|
| [e.g., Privilege escalation] | [High/Medium/Low] | [High/Medium/Low] | [e.g., Least-privilege IAM roles] |
| [e.g., Service impersonation] | [High/Medium/Low] | [High/Medium/Low] | [e.g., Mutual TLS, service mesh] |

### Trust Boundary 3: Service → Data/Infra

| Threat | Impact | Likelihood | Mitigation |
|--------|--------|------------|------------|
| [e.g., Data exfiltration] | [High/Medium/Low] | [High/Medium/Low] | [e.g., Encryption at rest, VPC endpoints] |
| [e.g., IAM role compromise] | [High/Medium/Low] | [High/Medium/Low] | [e.g., Permission boundaries, scoped policies] |

## IAM / Execution Role Blast Radius

| Role | Permissions | Blast Radius if Compromised | Mitigation |
|------|------------|---------------------------|------------|
| [e.g., Service execution role] | [e.g., S3 read/write, DynamoDB full] | [e.g., Read all user data, modify records] | [e.g., Scope to specific bucket/table, add permission boundary] |

## Error Handling & Information Leakage

| Component | Risk | Mitigation |
|-----------|------|------------|
| [e.g., API error responses] | [e.g., Stack traces expose internal paths] | [e.g., Generic error messages externally, structured logging internally] |
| [e.g., Log output] | [e.g., Secrets or PII in logs] | [e.g., Scrub sensitive fields before logging] |

## Runtime Security

| Component | Risk | Mitigation |
|-----------|------|------------|
| [e.g., Container runtime] | [e.g., Container escape, privilege escalation] | [e.g., Read-only filesystem, non-root user, seccomp profile] |
| [e.g., Network egress] | [e.g., SSRF, data exfiltration via outbound calls] | [e.g., Egress filtering, allowlisted destinations] |

## Secrets Lifecycle

| Secret | Provisioning | Rotation | Blast Radius if Leaked |
|--------|-------------|----------|----------------------|
| [e.g., Database credentials] | [e.g., Secrets Manager, injected at runtime] | [e.g., 90-day rotation, automated] | [e.g., Full database access — scope to specific tables] |
| [e.g., API keys] | [e.g., Environment variable] | [e.g., Manual] | [e.g., Third-party service abuse — add IP allowlist] |

## Data Lifecycle

| Data Type | At Rest | In Transit | Deletion | Access Control |
|-----------|---------|-----------|----------|---------------|
| [e.g., User PII] | [e.g., Encrypted, S3 SSE-KMS] | [e.g., TLS 1.3] | [e.g., Hard delete after 30 days] | [e.g., Service role only] |

## Supply Chain

| Dependency | Risk | Mitigation |
|-----------|------|------------|
| [e.g., npm packages] | [e.g., Malicious package update] | [e.g., Lock file, SCA scanning, pin versions] |
| [e.g., Base container image] | [e.g., Compromised upstream image] | [e.g., Pin digest, scan with Trivy] |
| [e.g., CI/CD actions] | [e.g., Compromised third-party action] | [e.g., Pin to commit SHA, audit action source] |
| [e.g., IaC modules] | [e.g., Malicious Terraform module] | [e.g., Pin version, internal module registry] |
| [e.g., LLM-generated code] | [e.g., Model suggests vulnerable patterns] | [e.g., SAST scanning, human review of security-critical paths] |

## Gate Questions

- [ ] What is the worst thing an adversary can do at each trust boundary?
- [ ] If the execution role is compromised, what is the blast radius?
- [ ] Does the infrastructure have the same threat coverage as the application code?

---

*This file is the sole input to Phase 5. Everything not written here does not carry forward.*
