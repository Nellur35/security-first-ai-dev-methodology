# Phase 3 -- Architecture & Design: CloudJanitor

**Project:** CloudJanitor -- AI-powered AWS security configuration agent
**Distribution:** AMI on AWS Marketplace
**Date:** 2025-01-15

---

## Problem Context (from Phase 2)

CloudJanitor assesses an AWS account's security posture, recommends remediation via curated recipes, deploys those recipes as CloudFormation stacks, and verifies compliance. It ships as a self-contained AMI that customers launch into their own VPC. The Brain Agent (Claude via Bedrock) orchestrates the workflow; humans interact through a NiceGUI dashboard over HTTPS.

---

## Component Architecture

```
                  Customer VPC (Private Subnet)
    ┌──────────────────────────────────────────────────────────────┐
    │                                                              │
    │   ┌────────────────────────────────────────────────────┐     │
    │   │              EC2 Instance (AMI)                     │     │
    │   │                                                    │     │
    │   │   ┌──────────────┐     ┌──────────────────────┐   │     │
    │   │   │  Controller  │────▶│  Brain Agent          │   │     │
    │   │   │  (systemd)   │◀────│  (Claude via Bedrock) │   │     │
    │   │   └──────┬───────┘     └──────────────────────┘   │     │
    │   │          │                                         │     │
    │   │   ┌──────┴───────┐                                 │     │
    │   │   │  Dashboard   │  :443 ◀── User (HTTPS)          │     │
    │   │   │  (NiceGUI)   │                                 │     │
    │   │   └──────────────┘                                 │     │
    │   └────────────────────────────────────────────────────┘     │
    │          │                                                   │
    │          │  VPC Endpoints (no public internet)                │
    │          ▼                                                   │
    │   ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐    │
    │   │ Lambda:      │  │ Lambda:      │  │ Lambda:          │    │
    │   │ assess_      │  │ deploy_      │  │ check_           │    │
    │   │ environment  │  │ recipe       │  │ compliance       │    │
    │   └──────┬───────┘  └──────┬───────┘  └──────┬───────────┘   │
    │          │                 │                  │               │
    │          ▼                 ▼                  ▼               │
    │   ┌──────────┐     ┌──────────────┐   ┌──────────────┐       │
    │   │ AWS APIs │     │ S3 Bucket    │   │ CloudFormation│       │
    │   │ (read)   │     │ (recipes +   │   │ (stack status)│       │
    │   └──────────┘     │  templates)  │   └──────────────┘       │
    │                    └──────────────┘                           │
    └──────────────────────────────────────────────────────────────┘
```

---

## Components

### 1. EC2 Instance (AMI)

The distribution unit. Built with Packer from Amazon Linux 2023, CIS Level 1 hardened. Contains two systemd services (Controller and Dashboard), Python 3.12 runtime, and a self-signed TLS certificate replaced at deployment. Runs as a dedicated `cloudjanitor` system user with `NoNewPrivileges`, `ProtectSystem=strict`, and `PrivateTmp` enforced by systemd.

**Boundaries:** The instance is the trust perimeter. All AWS API access flows through an EC2 instance role. No credentials are stored on disk. Configuration is pulled from SSM Parameter Store at boot via UserData.

### 2. Controller Service

Orchestration process running as `cloudjanitor-controller.service`. Reads configuration from `/opt/cloudjanitor/config/settings.yaml` (populated at boot from SSM parameters). Responsible for invoking the Brain Agent, dispatching Lambda tool calls, and managing correlation IDs across the assess-deploy-compliance flow.

**Interface:**
- Input: Configuration from SSM, user requests from Dashboard
- Output: Structured JSON results back to Dashboard, structured logs to CloudWatch

### 3. Brain Agent (Claude via Bedrock)

The reasoning component. Invoked via `bedrock:InvokeModel` through a VPC endpoint. The Brain Agent receives the assessment report and produces a remediation plan -- selecting recipes, parameterizing deployments, and interpreting compliance results.

**Interface:**
- Input: Assessment JSON, available recipes, compliance reports
- Output: Structured tool-use calls (assess, deploy, check compliance), natural language explanations for the Dashboard

**Critical constraint:** The Brain Agent has no direct AWS API access. It can only act through the three Lambda tools. This is the primary architectural control against prompt injection -- even if the agent is manipulated, its blast radius is limited to what the Lambdas allow.

### 4. Dashboard (NiceGUI)

Web interface running as `cloudjanitor-dashboard.service` on port 443 with TLS. Depends on Controller. Presents assessment results, recipe recommendations, deployment status, and compliance scores. The security group restricts access to a customer-specified CIDR block.

**Interface:**
- Input: User interaction (HTTPS, customer-specified CIDR only)
- Output: Rendered security posture, deployment controls

### 5. Lambda Tools

Three Lambda functions, each with a single responsibility. All share the same execution role (`CloudJanitor-LambdaRole`) with least-privilege policies scoped to the recipe S3 bucket, SSM parameters, and cross-account role assumption.

#### 5a. assess_environment

Read-only scan of the customer's AWS security posture. Checks 9 services: CloudTrail, GuardDuty, SecurityHub, Config, IAM (password policy, MFA, access keys), VPC flow logs, S3 public access block, KMS key rotation, EBS default encryption. Each check is fault-tolerant -- one failure does not abort the others. Produces recommendations mapped to recipe IDs.

**Interface:**
- Input: `{ correlation_id, checks[] }`
- Output: `{ statusCode, body: { checks: {}, summary: {}, recommended_recipes: [] } }`

#### 5b. deploy_recipe

Fetches a recipe YAML from S3, validates it, resolves the corresponding CloudFormation template, and creates or updates the stack. Supports dry-run validation. Tags every stack with `ManagedBy: CloudJanitor` and the recipe ID/version.

**Interface:**
- Input: `{ recipe_id, bucket, correlation_id, dry_run, parameters, wait }`
- Output: `{ statusCode, body: { action, status, stack_name, correlation_id } }`

#### 5c. check_compliance

Queries AWS Config rule compliance, aggregates SecurityHub findings by severity, and runs custom validation checks defined in the recipe's `validation` array. Produces a compliance score (percentage + rating: EXCELLENT/GOOD/FAIR/POOR/CRITICAL).

**Interface:**
- Input: `{ recipe_id, bucket, correlation_id, severity_filter }`
- Output: `{ statusCode, body: { score: {}, config_compliance: {}, securityhub_findings: {}, recipe_validations: [] } }`

### 6. S3 Bucket

`cloudjanitor-recipes-{AccountId}-{Region}`. Stores recipe YAML files under `recipes/` and CloudFormation templates under `cloudformation/`. Versioning enabled, KMS encryption, all public access blocked, TLS-only bucket policy. Retained on stack deletion.

### 7. CloudFormation

All infrastructure deployments are CloudFormation stacks. Recipes map 1:1 to templates. Stack names follow the pattern `cj-{recipe-id}`. Stack status is the source of truth for deployment state -- there is no separate state store.

### 8. Infrastructure (CloudFormation)

The master `infrastructure/template.yaml` deploys:
- S3 recipe bucket
- EC2 instance role + instance profile
- Lambda execution role
- Security group (443 only, customer-specified CIDR)
- VPC endpoints (Bedrock, Lambda, S3, CloudWatch Logs, SSM, SSM Messages, STS)
- SSM parameters for configuration
- EC2 instance with UserData

---

## Dependency Injection & Client Factories

All AWS API access goes through a `_get_client(service, region)` factory function. In the shared library (`cloudjanitor/lib/aws_helpers.py`), this is `get_client()` with a default retry configuration (3 attempts, adaptive mode, 5s connect timeout, 30s read timeout). Each Lambda handler has its own `_get_client()` with the same pattern.

This design exists for testability. In tests, `moto`'s `@mock_aws` decorator intercepts `boto3.client()` calls, so no patching of client factories is needed. The factory pattern also ensures consistent retry behavior and timeout configuration across all components.

```python
# Production: uses real boto3 client with retry config
def _get_client(service: str, region: str | None = None) -> Any:
    kwargs: dict[str, Any] = {"config": RETRY_CONFIG}
    if region:
        kwargs["region_name"] = region
    return boto3.client(service, **kwargs)

# Tests: moto intercepts at the boto3 layer
@mock_aws
def test_check_cloudtrail():
    # boto3.client("cloudtrail") returns moto's mock
    result = _check_cloudtrail()
    assert result["status"] == "not_configured"
```

---

## Testability Design

### Unit Tests

Every Lambda handler is tested with `moto` for AWS mocking. Test credentials are injected via `conftest.py` fixtures (`AWS_ACCESS_KEY_ID=testing`, `AWS_DEFAULT_REGION=us-east-1`). The `@mock_aws` decorator provides a clean, isolated AWS account for each test.

Key test fixtures:
- `sample_recipe`: Minimal valid recipe dictionary
- `sample_recipe_with_validation`: Recipe with compliance check definitions
- `s3_recipe_bucket`: Pre-populated moto S3 bucket with test recipe and CFN template

### Integration Tests

The full assess-deploy-compliance flow runs against moto. The `TestFullFlow` class sets up an S3 bucket with a test recipe and CFN template, then runs all three Lambda handlers in sequence with a shared correlation ID.

### What Cannot Be Tested with Moto

- Brain Agent (Bedrock) interactions: requires mock/stub at the Controller level
- NiceGUI Dashboard rendering: requires browser-level testing (out of scope for CI)
- Cross-account role assumption with ExternalId: moto supports basic STS but not full cross-account simulation
- VPC endpoint connectivity: infrastructure-level, validated by CloudFormation deployment

### Error Handling Pattern

All Lambda handlers follow the same pattern:
1. Known errors (e.g., `DeploymentError`, `RecipeValidationError`) return `statusCode: 400` with a structured `{ error, code, correlation_id }` body
2. Unknown exceptions return `statusCode: 500` with a generic message -- never a stack trace
3. Individual check functions within `assess_environment` catch their own exceptions so one failure does not block others

---

## Data Flows

### Assess-Deploy-Compliance Flow

```
User ──▶ Dashboard ──▶ Controller ──▶ Brain Agent (Bedrock)
                                          │
                          ┌────────────────┤
                          ▼                ▼
                    assess_environment   (reasoning)
                          │                │
                          ▼                ▼
                    recommendations ──▶ deploy_recipe
                                          │
                                          ▼
                                    CloudFormation stack
                                          │
                                          ▼
                                    check_compliance
                                          │
                                          ▼
                                    compliance score
                                          │
                                          ▼
                    Dashboard ◀── Controller ◀── Brain Agent
```

### Correlation ID

Every request generates a `correlation_id` (UUID) that flows through all three Lambda invocations and appears in every structured log entry. This is the primary debugging and audit mechanism.

### Recipe Lifecycle

```
Recipe YAML (S3) ──▶ Schema validation ──▶ CFN template resolution
                                                    │
                                                    ▼
                                          CloudFormation deploy
                                                    │
                                                    ▼
                                          Stack status polling
                                                    │
                                                    ▼
                                          Compliance check
                                              (recipe.validation[])
```

---

## Security Architecture Summary

| Boundary | Control |
|----------|---------|
| Internet to EC2 | Security group: port 443 only, customer CIDR |
| EC2 to AWS APIs | VPC endpoints only, no NAT gateway / internet gateway |
| EC2 to Bedrock | VPC endpoint, instance role `bedrock:InvokeModel` only |
| Brain Agent to Lambdas | Tool-use interface only -- agent cannot make arbitrary AWS calls |
| Lambdas to AWS APIs | Dedicated execution role with resource-scoped policies |
| S3 bucket | KMS encryption, versioning, public access blocked, TLS-only |
| CloudFormation stacks | Tagged `ManagedBy: CloudJanitor`, capabilities explicitly declared |
| OS level | Dedicated user, `NoNewPrivileges`, `ProtectSystem=strict`, CIS Level 1 hardening |

---

## Decisions & Rejected Alternatives

### Direct Lambda Invocation over Step Functions

**Chosen:** The Controller invokes Lambdas directly via `lambda:InvokeFunction`.

**Rejected:** AWS Step Functions for orchestrating the assess-deploy-compliance flow.

**Reason:** Step Functions add an abstraction layer that is difficult to test locally. The flow is linear (assess, then deploy, then check compliance) with no parallel branches or human-approval steps. The Brain Agent handles retry logic and decision-making, which duplicates what Step Functions would provide. Direct invocation is simpler to debug, simpler to test with moto, and does not require IAM permissions for `states:*` actions. If the flow becomes more complex (e.g., multi-account parallel deployments), this decision should be revisited.

### S3 + CloudFormation Stack Status over DynamoDB for State

**Chosen:** Recipe state lives in S3 (recipe YAML + CFN template). Deployment state lives in CloudFormation stack status (`CREATE_COMPLETE`, `UPDATE_COMPLETE`, etc.).

**Rejected:** DynamoDB table for tracking deployment state, recipe metadata, and compliance history.

**Reason:** DynamoDB adds an operational dependency that the customer must manage (capacity, backup, costs). CloudFormation already tracks stack state with full event history. S3 already stores recipes with versioning. Adding DynamoDB would create a second source of truth that could diverge from the actual stack state. The "no extra state store" design means there is exactly one place to look for deployment status: the CloudFormation stack itself. Trade-off: historical compliance scores are not persisted across invocations. If trend reporting becomes a requirement, this decision should be revisited.

### Single EC2 Instance over ECS/EKS

**Chosen:** Single EC2 instance running two systemd services.

**Rejected:** Containerized deployment on ECS Fargate or EKS.

**Reason:** CloudJanitor is sold as an AMI on AWS Marketplace. Marketplace AMI is the simplest distribution model for this product. ECS/EKS would require the customer to have container infrastructure, which many target customers (smaller teams adopting security tooling for the first time) do not have. A single EC2 instance is operationally simple, easy to debug via SSM Session Manager, and matches the "deploy one thing, get security" value proposition. Trade-off: no horizontal scaling. This is acceptable because CloudJanitor manages one account (or one organization) at a time and the workload is not latency-sensitive.

### NiceGUI over React/Vue SPA

**Chosen:** NiceGUI (Python-native web framework) for the Dashboard.

**Rejected:** React or Vue SPA with a separate API backend.

**Reason:** NiceGUI keeps the entire codebase in Python, eliminating a JavaScript build pipeline, a second language in the repo, and a second set of dependencies to audit. The Dashboard is an operational tool, not a customer-facing product UI. It needs to display assessment results, deployment status, and compliance scores -- all of which NiceGUI handles adequately. Trade-off: NiceGUI's component library is less mature than React's. This is acceptable for an internal operational dashboard.

### VPC Endpoints over NAT Gateway

**Chosen:** VPC endpoints for all AWS service access (Bedrock, Lambda, S3, CloudWatch Logs, SSM, STS).

**Rejected:** NAT gateway for outbound internet access.

**Reason:** A NAT gateway allows the instance to reach any internet endpoint, which vastly increases the attack surface if the instance is compromised. VPC endpoints restrict API access to specific AWS services with no path to the public internet. This is more expensive (7 interface endpoints at ~$7.30/month each) but eliminates an entire class of exfiltration attacks. The instance has no legitimate need for internet access -- all its dependencies are AWS services.

### Structured JSON Logging over Plaintext

**Chosen:** `structlog` with JSON rendering and correlation IDs in every log entry.

**Rejected:** Python standard `logging` module with plaintext format.

**Reason:** JSON logs are parseable by CloudWatch Logs Insights, enabling queries like "show all events for correlation_id X." Correlation IDs across the three-Lambda flow would be impossible to trace with unstructured logs. `structlog` also enforces structured event names (`assess_environment.check.start`) that are easier to alert on than free-text messages.

### Recipe Schema Validation with jsonschema over Ad-Hoc Checks

**Chosen:** JSON Schema (defined in `recipes/schema.yaml`) validated by the `jsonschema` Python library.

**Rejected:** Ad-hoc validation with if-statements in each handler.

**Reason:** A schema is a single source of truth for recipe structure that can be validated in CI (Job 6: recipe-validation), in Lambda handlers, and by external tools. Ad-hoc validation scatters the rules across multiple files, making it easy to miss a constraint when adding a new recipe field. The schema also documents the recipe format for customers who want to write custom recipes. The `additionalProperties: false` constraint prevents silent data loss from typos in recipe fields.
