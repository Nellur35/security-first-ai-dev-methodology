# Phase 2 — Product Requirements: CloudJanitor Brain Agent

**Input:** Phase 1 problem statement

---

## Functional Requirements

### Core Reasoning Flow

- **REQ-F001:** Accept natural language customer requests. Maximum input size: 2000 characters.
  - *Test: Assert requests >2000 chars are rejected with clear error; valid requests are accepted.*

- **REQ-F002:** Classify request complexity into Quick (single known recipe), Standard (multi-recipe/compliance goals), Full (complex/multi-org), or Builder (custom recipe creation). Deterministic classifier with <5% misclassification rate on representative test set.
  - *Test: Assert correct classification for 50+ labeled test cases; measure error rate.*

- **REQ-F003:** Invoke `assess_environment` Lambda before making recommendations. Treat assessment output as untrusted data — validate structure, enforce size limits, sanitize before embedding in prompts.
  - *Test: Mock assess_environment; assert called once per request; assert malformed output is rejected gracefully.*

- **REQ-F004:** Apply the appropriate reasoning pipeline variant based on complexity:
  - Quick: Direct recipe lookup (no pipeline)
  - Standard: RCA → ToT → Pre-Mortem
  - Full: FP → RCA → Adversarial → ToT → Pre-Mortem
  - Builder: Full + Validation
  - *Test: For each level, assert reasoning trace includes expected stage markers.*

- **REQ-F005:** Recommend 1-5 recipes with: recipe name/ID, justification, estimated cost (from recipe metadata, not LLM-generated), compliance mappings (from recipe metadata), deployment order/dependencies.
  - *Test: Assert recommendation schema contains all fields; verify cost and compliance values match recipe catalog (not LLM output).*

- **REQ-F006:** Include a reasoning trace in every response showing which stages executed and key insights from each.
  - *Test: Assert response contains structured trace with stage labels; verify trace is non-empty.*

- **REQ-F007:** Recommend recipes only. Do not deploy without explicit, authenticated customer approval. See REQ-F019 for approval mechanism.
  - *Test: Assert deploy_recipe is never called without valid approval token.*

- **REQ-F008:** After deployment, invoke `check_compliance` and report compliance score delta (before/after).
  - *Test: Assert check_compliance called after deploy_recipe; verify delta calculation.*

### Approval and Authorization

- **REQ-F019:** Customer approval must be a signed, non-replayable token that specifies exactly which recipe IDs and versions are approved for deployment. Approval tokens expire after 15 minutes.
  - *Test: Assert expired tokens are rejected; assert token for Recipe A cannot deploy Recipe B; assert replayed tokens are rejected.*

- **REQ-F020:** Define access model: who can invoke the Brain Agent, how they are authenticated, and what actions they are authorized to perform.
  - *Test: Assert unauthenticated requests are rejected; assert authorization checks run before any Lambda invocation.*

- **REQ-F021:** Rate limit requests: maximum 10 concurrent requests per customer account, maximum 100 requests per hour.
  - *Test: Assert request 11 is queued or rejected; assert request 101 in an hour returns rate limit error.*

### Safety and Validation

- **REQ-F009:** Reject unvalidated custom recipes. Builder pipeline calls validate_recipe (deterministic) before any deployment.
  - *Test: Assert custom recipes without validation are rejected.*

- **REQ-F010:** Never expose system prompt content to customers.
  - *Test: Search all response text for system prompt fragments; assert zero matches.*

- **REQ-F011:** Never expose stack traces or internal error details to customers. Return generic error with correlation ID.
  - *Test: Simulate Lambda failures; assert response contains generic message, not stack trace.*

- **REQ-F012:** Defend against prompt injection using layered controls:
  1. Input pattern scanning (known injection patterns — necessary but not sufficient)
  2. Environment data sanitization (assess_environment output treated as untrusted)
  3. Structured output validation (recipe IDs must exist in catalog; costs from metadata, not LLM)
  4. Deployment boundary validation (deploy_recipe independently verifies recipe ID and parameters)
  - *Test: Assert known injection patterns rejected; assert injected environment data does not alter recommendations; assert non-existent recipe IDs are filtered; assert deploy_recipe rejects unrecognized parameters.*

- **REQ-F022:** Validate all LLM output against a strict schema before acting on it. Recipe IDs must exist in catalog. Costs and compliance mappings must come from recipe metadata, not from LLM-generated text.
  - *Test: Mock Bedrock to return hallucinated recipe IDs; assert they are filtered out. Mock Bedrock to return wrong costs; assert catalog costs are used instead.*

### Integration

- **REQ-F013:** Call Amazon Bedrock via VPC endpoint. Pin to a specific model version (e.g., `anthropic.claude-sonnet-4-5-v2:1`). No internet egress.
  - *Test: Assert invoke_model uses correct model version; assert endpoint URL contains vpce-.*

- **REQ-F014:** Retrieve system prompt from Secrets Manager on cold start. Cache in a defensive wrapper that prevents serialization, logging, or string conversion outside the Bedrock invocation path.
  - *Test: Assert get_secret_value called on init; assert str(prompt_wrapper) raises exception; assert prompt not in logs.*

- **REQ-F015:** Log all requests and responses with correlation IDs via structlog (structured JSON). Correlation ID must propagate across all Lambda invocations.
  - *Test: Trace correlation ID through all 4 Lambda log groups; assert it appears in all.*

### Performance

- **REQ-F016:** p95 latency < 45 seconds for Standard complexity. **Requires Phase 3.5 spike validation before committing.**
  - *Test: Load test with 100 Standard requests; assert p95 < 45s.*

- **REQ-F017:** p95 latency < 120 seconds for Full complexity.
  - *Test: Load test with 50 Full requests; assert p95 < 120s.*

- **REQ-F018:** Handle Bedrock throttling with exponential backoff (max 3 retries, max 30s total backoff).
  - *Test: Mock ThrottlingException; assert retry logic; verify max 3 attempts.*

---

## Non-Functional Requirements

### Security

- **REQ-NF001:** Runs entirely in customer's AWS account. No cross-account access to CloudJanitor infrastructure.
  - *Test: Verify Lambda template has no cross-account IAM roles.*

- **REQ-NF002:** Least-privilege IAM: only specific Lambda ARNs (assess_environment, deploy_recipe, check_compliance), specific Secrets Manager secret ARN, specific Bedrock model ARN.
  - *Test: IAM Policy Simulator — verify role cannot perform ec2:TerminateInstances, s3:DeleteBucket, iam:CreateUser.*

- **REQ-NF003:** System prompt encrypted in Secrets Manager with customer-managed KMS key.
  - *Test: Assert secret has KMS key ARN; verify key is customer-managed.*

- **REQ-NF004:** All Bedrock API calls via VPC endpoint. No public internet egress.
  - *Test: VPC Flow Logs — assert zero public internet connections to Bedrock.*

- **REQ-NF005:** No persistent storage of customer requests or Bedrock responses. CloudWatch Logs only.
  - *Test: Verify Lambda code has no S3/DynamoDB/RDS writes.*

- **REQ-NF014:** Recipe CloudFormation templates must be integrity-verified before deployment (checksums or signatures). Recipe templates must pass cfn-lint and cfn-nag scanning.
  - *Test: Assert deployment rejected if template checksum doesn't match catalog; assert cfn-lint/cfn-nag runs before deploy.*

### Reliability

- **REQ-NF006:** 99.5% structured response rate — defined as: response conforms to JSON schema, all recipe IDs exist in catalog, reasoning trace present, latency within target for complexity class.
  - *Test: Chaos testing — inject failures; assert >99.5% return schema-valid JSON.*

- **REQ-NF007:** Graceful degradation if assess_environment fails: return baseline recommendations (CloudTrail + IAM Hardening) with explicit disclaimer "assessment unavailable — recommendations are general."
  - *Test: Mock assess_environment failure; assert response contains recommendations + warning.*

- **REQ-NF008:** 20-second timeout per reasoning stage. Partial results acceptable — pipeline continues with available reasoning.
  - *Test: Mock Bedrock delay 25s; assert timeout triggers; pipeline continues.*

- **REQ-NF015:** Rollback strategy: capture pre-deployment state; if compliance score decreases or deployment fails, trigger CloudFormation stack rollback. Report rollback status to customer.
  - *Test: Mock deployment that decreases compliance score; assert rollback triggered; assert customer notified.*

### Observability

- **REQ-NF009:** Correlation IDs in all Lambda invocations (Brain Agent, assess, deploy, compliance).
  - *Test: CloudWatch Logs Insights query across all 4 log groups; assert ID appears in all.*

- **REQ-NF010:** CloudWatch metrics: request_count, recommendation_count, bedrock_token_count, bedrock_latency_ms, error_count (by type), cost_per_request_usd.
  - *Test: Assert namespace CloudJanitor/BrainAgent contains all metrics.*

- **REQ-NF011:** Log reasoning pipeline stage transitions (stage name, status, duration_ms).
  - *Test: Assert log entries like "stage=root_cause_analysis status=complete duration_ms=3421".*

### Cost Control

- **REQ-NF012:** Enforce per-request cost cap of $5. Estimate cost before each Bedrock call based on prompt token count. Abort pipeline if projected total exceeds cap.
  - *Test: Construct prompt exceeding $5 projected cost; assert request aborted before Bedrock call.*

- **REQ-NF013:** Use Bedrock streaming API where possible.
  - *Test: Assert invoke_model_with_response_stream is used.*

- **REQ-NF016:** Aggregate cost tracking: daily and monthly per-customer caps configurable via Parameter Store.
  - *Test: Assert requests rejected when daily cap reached; assert cap values read from Parameter Store.*

### Versioning

- **REQ-NF017:** Track recipe versions. Environment assessment must include currently-deployed recipe versions. Brain Agent must be aware of version compatibility.
  - *Test: Assert recommendations include recipe version; assert version conflict detected when upgrading.*

- **REQ-NF018:** Pin Bedrock model version. Behavioral regression test suite must pass before any model version upgrade.
  - *Test: Assert model version in invoke_model matches pinned version; assert regression suite exists and runs.*

---

## Explicit Exclusions

- **EX-001:** No custom recipe creation from scratch — Builder pipeline assists but requires human validation
- **EX-002:** No autonomous deployment — always requires authenticated customer approval
- **EX-003:** No real-time monitoring or alerting — recommends monitoring recipes; actual alerting via CloudWatch/GuardDuty
- **EX-004:** No multi-cloud — AWS only
- **EX-005:** No incident response — preventative configuration tool, not IR
- **EX-006:** No model fine-tuning — uses Bedrock foundation models as-is
- **EX-007:** No conversation history — stateless per-request design (state for multi-step approval flow managed via signed tokens, not server-side persistence)

---

## Definition of Done

### Categorical Acceptance Criteria

1. For any request within recipe catalog scope, the system returns at least one relevant recommendation
2. Every recommendation includes cost (from catalog), compliance mapping (from catalog), reasoning trace, and deployment order
3. Approval-to-deployment completes within 60 seconds for up to 4 recipes
4. Compliance score delta is reported accurately
5. Zero prompt injection bypasses (penetration tested with 50+ attack patterns including indirect injection via environment data)
6. Zero system prompt leakage under adversarial testing
7. Full CI/CD pipeline passes (unit + integration + E2E)
8. SAST, SCA, secret scanning, IaC scanning gates pass
9. 10 beta customers complete a real compliance workflow without support escalation

### Example Scenario (one of many test cases)

A security engineer types: "Help me pass SOC2 Type 2 audit — we need to close findings in access control, logging, and encryption."

The Brain Agent assesses their account, returns 4 recipe recommendations with SOC2 control mappings, estimated cost ($47/month), deployment order, and a Pre-Mortem warning ("if monitoring is deployed before CloudTrail, alarms will fire immediately — recommend 24-hour soak period"). The engineer approves, recipes deploy in order, compliance score increases. The engineer can explain to their CISO exactly why these controls were chosen.

---

## Decisions & Rejected Alternatives

| Decision | Alternative Rejected | Reason |
|----------|---------------------|--------|
| Bedrock foundation model (no fine-tuning) | Fine-tune on recipe corpus | Ongoing training data curation cost; system prompt + structured prompts sufficient for reasoning |
| System prompt encrypted in Secrets Manager | Embed in Lambda code | If Lambda code is exposed, system prompt (trade secret + guardrails) is compromised |
| Stateless per-request with signed approval tokens | Session state in DynamoDB | Simpler, more secure; signed tokens carry approval context without server-side persistence |
| Reasoning trace always included | Optional trace for faster responses | Trace is core value proposition — transparency on why recommendations were made |
| Deterministic complexity router | LLM self-classification | 10x faster, zero tokens; LLM invoked after classification, not before |
| VPC endpoint for Bedrock | Public internet egress | Customer data stays within VPC; meets compliance requirements |
| Layered prompt injection defense | Pattern blocklist only | Pattern matching is necessary but not sufficient; structural controls (output validation, deployment boundary checks) are the real defense |
| Cost estimation before Bedrock call | Post-hoc cost tracking only | Must prevent cost overruns, not just report them |
| Recipe template integrity verification | Trust templates from S3 | Templates modify production infrastructure — must be verified before deployment |
| Explicit rollback mechanism | Manual recovery | System deploys infrastructure; must be able to undo it |
