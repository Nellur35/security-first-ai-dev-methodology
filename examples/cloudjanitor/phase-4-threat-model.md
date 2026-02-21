# Phase 4 -- Threat Model: CloudJanitor

**Project:** CloudJanitor -- AI-powered AWS security configuration agent
**Distribution:** AMI on AWS Marketplace
**Date:** 2025-01-16
**Input:** `phase-3-architecture.md`

---

## Why This Phase Exists

CloudJanitor is a security product that has privileged access to its customer's AWS account. It can create CloudFormation stacks, modify IAM password policies, enable security services, and read sensitive configuration data. If this system is compromised, the attacker does not just own the CloudJanitor instance -- they own the customer's AWS infrastructure.

Every architectural decision in Phase 3 was made assuming a cooperative world. This phase injects adversarial thinking. The goal is to identify what an attacker sees, what they can manipulate, and what the worst outcome is -- then design mitigations before a single line of implementation code is written.

---

## Trust Boundaries

### Trust Boundary 1: User to Dashboard

```
User (browser) ──HTTPS──▶ Dashboard (NiceGUI, port 443)
```

**What an adversary sees:** A web application accessible from a CIDR range. The self-signed TLS certificate in the AMI means the first connection has no trust anchor until the customer replaces it.

**Threats:**

| Threat | Impact | Likelihood | Mitigation |
|--------|--------|------------|------------|
| Unauthorized access to Dashboard | Full control of CloudJanitor: trigger assessments, deploy recipes, view security posture | Medium | Security group restricts to customer-specified CIDR. No default 0.0.0.0/0. Dashboard must implement authentication (not just network ACL). |
| Man-in-the-middle on first connection | Attacker intercepts session, gains dashboard access | Low (requires VPC-level access) | Self-signed cert is a known weakness. Documentation must instruct customers to replace with ACM or their own CA cert. SSM Session Manager is the recommended management path. |
| Session hijacking | Attacker reuses stolen session token | Low | NiceGUI sessions are server-side WebSocket connections. No persistent cookie tokens by default. However, session timeout must be explicitly configured -- NiceGUI does not enforce one. |
| XSS / injection through Dashboard inputs | Execute arbitrary JavaScript in other users' browsers | Low (single-user tool) | NiceGUI server-side rendering limits client-side attack surface. Input validation required on any user-provided parameters (recipe parameters, CIDR inputs). |

**Residual risk:** Dashboard authentication is currently network-level only (security group CIDR). This is insufficient for multi-user environments. An authentication mechanism (at minimum HTTP Basic Auth with a customer-set password via SSM parameter) should be added before multi-user support.

---

### Trust Boundary 2: Brain Agent to Lambda Tools

```
Brain Agent (Claude via Bedrock) ──tool_use──▶ Controller ──invoke──▶ Lambda
```

**What an adversary sees:** An LLM with tool-use capabilities that can invoke three Lambda functions. If the LLM can be manipulated into calling tools with malicious parameters, those parameters flow directly into AWS API calls.

**Threats:**

| Threat | Impact | Likelihood | Mitigation |
|--------|--------|------------|------------|
| Prompt injection via assessment data | Attacker places malicious content in AWS resource names/tags that gets included in the assessment report fed to the Brain Agent. Agent is manipulated into deploying a malicious recipe or ignoring findings. | Medium | The Brain Agent receives structured JSON, not raw text. Assessment data should be sanitized before inclusion in prompts. Resource names and tags should be treated as untrusted input. The Controller must validate tool-use calls before execution -- the agent's output is a suggestion, not a command. |
| Recipe ID injection | Agent is manipulated into requesting deployment of a recipe with a crafted ID (e.g., `../../etc/passwd` or `; rm -rf /`) | Low | Recipe IDs are validated against the schema pattern `^[a-z0-9]+(-[a-z0-9]+)*$`. The `_fetch_recipe()` function constructs the S3 key as `recipes/{recipe_id}.yaml` -- path traversal is not possible in S3 keys used via the SDK. Lambda handler validates `recipe_id` is non-empty. |
| Arbitrary parameter injection | Agent passes unexpected CloudFormation parameters that change stack behavior | Medium | `_prepare_parameters()` merges recipe config and user parameters. The CFN template defines its own `Parameters` section with `AllowedValues` and `AllowedPattern` constraints. Parameters not declared in the template are rejected by CloudFormation. However, the `user_params` dict from the event is passed through without validation. This should be restricted to a known set per recipe. |
| Denial of service via repeated invocations | Agent enters a loop, invoking Lambdas continuously | Low | Lambda concurrent execution limits and Bedrock invocation rate limits provide natural throttling. The Controller should implement a per-session invocation counter with a hard ceiling. |

**Critical design decision from Phase 3 validated here:** The Brain Agent has no direct AWS API access. It can only act through the three Lambdas. This means even a fully compromised agent (complete prompt injection takeover) can only: (1) read security posture, (2) deploy recipes that exist in S3 with templates that exist in S3, and (3) check compliance. It cannot exfiltrate data, create IAM users, or modify resources outside the CloudFormation stack pattern.

---

### Trust Boundary 3: Lambda to AWS APIs

```
Lambda ──boto3──▶ AWS APIs (CloudFormation, S3, STS, Config, SecurityHub, etc.)
```

**What an adversary sees:** Lambda functions with IAM roles that grant access to AWS APIs. If a Lambda is compromised (code injection, dependency hijack), the attacker inherits the Lambda execution role's permissions.

**Threats:**

| Threat | Impact | Likelihood | Mitigation |
|--------|--------|------------|------------|
| Lambda execution role is too broad | Compromised Lambda can access resources beyond what it needs | Medium | The `LambdaExecutionRole` in `infrastructure/template.yaml` currently has S3 access scoped to the recipe bucket, STS scoped to `CloudJanitorCrossAccountRole`, and SSM scoped to `/cloudjanitor/*`. This is well-scoped. However, `deploy_recipe` needs `cloudformation:*` on the stacks it creates, which is on the EC2 instance role, not the Lambda role. **Verify this is correct -- if Lambda also gets CFN permissions, scope them.** |
| Cross-account role assumption without ExternalId | Attacker who compromises the Lambda assumes roles in customer's other accounts | Low | The `cross-account-role.yaml` template requires `ExternalId` in the `sts:AssumeRole` condition. Without the correct ExternalId, assumption fails. ExternalId is stored in SSM Parameter Store (encrypted). |
| S3 object injection via Lambda | Attacker writes a malicious recipe or CFN template to S3 via the Lambda role | Medium | Lambda role has `s3:PutObject` on the recipe bucket. A compromised assess or compliance Lambda could write a malicious recipe. **Mitigation: restrict `s3:PutObject` to only the `deploy_recipe` Lambda, and only to the `reports/` prefix. assess and compliance Lambdas need read-only S3 access.** |

---

### Trust Boundary 4: S3 Recipes (Tampering)

```
S3 Bucket ◀── Recipe YAML files + CloudFormation templates
```

**What an adversary sees:** The S3 bucket contains the instructions (recipes) and the infrastructure definitions (CFN templates) that CloudJanitor will deploy into the customer's account. Control the bucket, control the infrastructure.

**Threats:**

| Threat | Impact | Likelihood | Mitigation |
|--------|--------|------------|------------|
| Recipe tampering | Attacker modifies a recipe to deploy malicious infrastructure | High (if bucket access is obtained) | S3 versioning is enabled. Bucket policy enforces TLS-only. KMS encryption at rest. Bucket is not publicly accessible. Schema validation catches structural anomalies. **However: schema validation does not check the semantic content of `services[].config` -- a recipe could have valid structure but malicious intent (e.g., setting `block_public_acls: false`).** |
| CFN template injection | Attacker replaces a CloudFormation template with one that creates IAM roles with `AdministratorAccess` | Critical | Templates are stored in S3 alongside recipes. If an attacker can write to `cloudformation/cj-*.yaml`, they control what gets deployed. **Mitigation: CFN template validation (cfn-lint in CI), restricted resource types in templates (see IaC section below), and S3 object lock for production templates.** |
| Recipe created by compromised Brain Agent | The agent creates a custom recipe that looks legitimate but contains a backdoor | Medium | The `created_by` field in the recipe schema has allowed values: `cloudjanitor`, `brain-agent`, `user`. Recipes created by `brain-agent` should undergo additional validation before deployment: human approval via Dashboard, or at minimum a dry-run CFN validation. |

---

## IAM Blast Radius Analysis

### EC2 Instance Role (CloudJanitor-InstanceRole)

This is the most privileged role in the system. If the EC2 instance is compromised, the attacker gets:

| Permission | Blast Radius | Worst Case |
|------------|-------------|------------|
| `bedrock:InvokeModel` | All Bedrock models in the region | Attacker runs arbitrary model invocations. Financial impact (Bedrock charges per token), potential data exfiltration via model prompts. Scope to specific model ARNs if possible. |
| `lambda:InvokeFunction` | `cloudjanitor-*` functions only | Attacker can invoke the three Lambda tools. This is the intended tool-use path. Blast radius is limited to what the Lambdas can do (see Lambda role analysis). |
| `s3:GetObject/PutObject/ListBucket/DeleteObject` | Recipe bucket only | Attacker can read, modify, and delete recipes and templates. This is the recipe tampering vector. |
| `cloudformation:CreateStack/UpdateStack/DeleteStack` | **All stacks in the account** (`Resource: '*'`) | **This is the critical finding.** An attacker can create arbitrary CloudFormation stacks with `CAPABILITY_IAM` and `CAPABILITY_NAMED_IAM`. This means they can create IAM roles with any permissions, deploy any AWS resource, and effectively take over the account. |
| `ssm:GetParameter/PutParameter` | `/cloudjanitor/*` parameters only | Attacker can read configuration (bucket name, alert email, version) and modify parameters that affect CloudJanitor behavior. |
| `sts:AssumeRole` | `CloudJanitorCrossAccountRole` in any account | Attacker can pivot to any account that has the cross-account role deployed. ExternalId is the only barrier, and it is readable from SSM. |
| `config:Describe*/Get*`, `securityhub:Get*/Describe*`, `guardduty:Get*/List*` | Read-only security service data | Information disclosure. Attacker learns the full security posture. |
| `logs:CreateLogGroup/Stream, PutLogEvents` | All log groups | Attacker can write misleading log entries. Cannot read existing logs. |

**Critical finding:** `cloudformation:CreateStack` with `Resource: '*'` combined with `CAPABILITY_IAM` means a compromised instance can deploy any IAM role in the account. This is the single highest-risk permission in the entire system.

**Mitigations:**

1. **Permission boundary on the instance role.** Attach an IAM permissions boundary that restricts what CloudFormation stacks deployed by this role can create. At minimum: deny `iam:CreateRole` unless the role has tag `ManagedBy: CloudJanitor`, deny `iam:AttachRolePolicy` for `AdministratorAccess` and `PowerUserAccess`.

2. **Resource-scoped CloudFormation permissions.** Change `Resource: '*'` to `arn:aws:cloudformation:*:*:stack/cj-*/*`. This restricts stack operations to stacks with the `cj-` prefix only.

3. **Scope Bedrock to specific model ARNs.** Change `bedrock:InvokeModel` from `Resource: '*'` to the specific model ARN(s) used by the Brain Agent.

4. **Separate Lambda invocation by function.** Instead of `cloudjanitor-*` wildcard, use explicit function ARNs. This prevents invocation of any future Lambda that happens to match the prefix.

### Lambda Execution Role (CloudJanitor-LambdaRole)

Lower privilege than the instance role. Scoped to S3 (recipe bucket), STS (cross-account role), and SSM (read-only).

**Finding:** Lambda role does not have `cloudformation:*` permissions. CloudFormation operations in `deploy_recipe` use the Lambda role for the API call, but the stack itself is created with the Lambda's execution context. **Verify this is the intended design -- if Lambda needs CFN permissions, they should be explicitly scoped to `cj-*` stacks only.**

---

## Supply Chain Analysis

### Python Dependencies

The AMI installs the following pip packages:

| Package | Purpose | Risk | Mitigation |
|---------|---------|------|------------|
| `boto3` | AWS SDK | Low (maintained by AWS, heavily audited) | Pin to specific minor version. Monitor AWS security bulletins. |
| `structlog` | Structured logging | Low (small, well-maintained) | Pin version. Review changelog on updates. |
| `pyyaml` | YAML parsing | Medium (historically had deserialization vulnerabilities) | Use `yaml.safe_load()` exclusively -- never `yaml.load()`. This is already enforced in the codebase. Pin version. |
| `jsonschema` | Recipe validation | Low | Pin version. |
| `nicegui` | Dashboard UI | Medium (larger dependency tree, pulls in FastAPI, uvicorn, web dependencies) | Pin version. This is the largest dependency surface. NiceGUI pulls in dozens of transitive dependencies. Run `pip-audit` on every CI run. |

**CI enforcement:** Job 4 (Security Scan) runs `pip-audit --desc` on every push. This catches known CVEs in pinned dependencies. Bandit SAST scan catches dangerous patterns (`yaml.load`, `eval`, `exec`, etc.).

**Risk accepted:** `pip-audit` uses `continue-on-error: true` in the CI pipeline. This means upstream CVEs that the team cannot fix (e.g., a vulnerability in `boto3` that requires an AWS patch) will not block the pipeline. This is a deliberate trade-off -- a waiver pattern decision documented in the CI configuration. When an actionable CVE is found, it should be addressed immediately; when it is upstream-only, the waiver is acceptable.

### AMI Supply Chain

| Component | Risk | Mitigation |
|-----------|------|------------|
| Base AMI (Amazon Linux 2023) | Low (Amazon-maintained) | Sourced from `owners = ["amazon"]` in Packer config. AMI ID validated by the Amazon data source filter. |
| AWS CLI v2 | Low | Downloaded from `awscli.amazonaws.com` (Amazon CDN). Installed via official installer. |
| Packer build scripts | Medium | `install.sh`, `harden-os.sh`, `configure.sh` are version-controlled. Any modification is visible in git diff. |
| Docker | Low-Medium | Installed from AL2023 default repositories. Docker daemon runs but is not currently used by CloudJanitor services (Controller and Dashboard run directly on Python). **If Docker is not needed, remove it to reduce attack surface.** |

---

## Error Handling Analysis

### Lambda Error Responses

All three Lambda handlers follow a two-tier error pattern:

```python
# Known errors: return structured error with code
except (DeploymentError, RecipeValidationError) as e:
    return {
        "statusCode": 400,
        "body": json.dumps({
            "error": str(e),        # Human-readable message
            "code": e.code,          # Machine-readable code
            "correlation_id": correlation_id,
        }),
    }

# Unknown errors: return generic message, NO stack trace
except Exception as e:
    log.error("deploy_recipe.unexpected_error", error=str(e))
    return {
        "statusCode": 500,
        "body": json.dumps({
            "error": "Internal error during deployment",
            "code": "INTERNAL_ERROR",
            "correlation_id": correlation_id,
        }),
    }
```

**What we caught in testing:** The `assess_environment` handler's individual check functions catch `ClientError` and return `{"status": "error", "error": str(e)}`. The `str(e)` representation of a `ClientError` includes the full AWS error message, which can contain:
- Account IDs
- Resource ARNs
- IAM role names
- Region information

**Mitigation:** In the `except ClientError` blocks within check functions, log the full error server-side (`log.error(...)`) but return only the error code to the caller:

```python
# Current (leaks information):
except ClientError as e:
    return {"status": "error", "error": str(e)}

# Should be:
except ClientError as e:
    log.error("check_name.failed", error=str(e), error_code=e.response["Error"]["Code"])
    return {"status": "error", "error_code": e.response["Error"]["Code"]}
```

This applies to all 9 check functions in `assess_environment` and the compliance check functions.

### Brain Agent Error Handling

The Brain Agent (Claude via Bedrock) can produce malformed tool-use calls, hallucinate recipe IDs, or return non-JSON output. The Controller must:
1. Validate every tool-use call against expected schemas before invoking Lambda
2. Handle Bedrock throttling (429) and service errors (5xx) with retry + backoff
3. Never pass raw model output to the Dashboard without sanitization
4. Implement a maximum conversation turn limit to prevent infinite loops

---

## Infrastructure as Code Threat Analysis

### CloudFormation Templates as Attack Surface

CloudFormation templates are the most dangerous artifact in the system. A template defines what AWS resources to create, including IAM roles, security groups, Lambda functions, and custom resources with arbitrary code. CloudJanitor deploys templates from S3 with `CAPABILITY_IAM` and `CAPABILITY_NAMED_IAM`.

**Threat scenario:** An attacker (or a manipulated Brain Agent) creates a recipe that maps to a CloudFormation template containing:

```yaml
Resources:
  BackdoorRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: LegitLookingServiceRole
      AssumeRolePolicyDocument:
        Statement:
          - Effect: Allow
            Principal:
              AWS: arn:aws:iam::ATTACKER_ACCOUNT:root
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AdministratorAccess
```

This template creates an IAM role that grants administrator access to an external account. It would pass `cfn-lint` syntax validation. It would pass schema validation (the schema validates recipes, not templates). It would be deployed by `deploy_recipe` without question.

**Mitigations:**

1. **Restricted resource types.** The `deploy_recipe` handler should maintain an allowlist of permitted CloudFormation resource types. For security recipes, the allowed types should be:
   - `AWS::CloudTrail::Trail`
   - `AWS::GuardDuty::Detector`
   - `AWS::SecurityHub::Hub`, `AWS::SecurityHub::Standard`
   - `AWS::Config::ConfigurationRecorder`, `AWS::Config::DeliveryChannel`
   - `AWS::AccessAnalyzer::Analyzer`
   - `AWS::SNS::Topic`, `AWS::SNS::Subscription`
   - `AWS::S3::Bucket`, `AWS::S3::BucketPolicy`
   - `AWS::IAM::Role` (only with tag `ManagedBy: CloudJanitor`)
   - `AWS::KMS::Key`
   - `AWS::Lambda::Function` (only for custom resources within the template)
   - `AWS::CloudFormation::WaitConditionHandle`

   Any template containing resource types outside this list should be rejected.

2. **Template content scanning.** Before deployment, parse the template YAML and check:
   - No IAM policies with `Effect: Allow` on `Resource: '*'` and `Action: '*'`
   - No `AssumeRolePolicyDocument` that references external account IDs
   - No `AWS::IAM::ManagedPolicy` resources with admin-level permissions
   - No `AWS::EC2::SecurityGroup` with `0.0.0.0/0` ingress on non-443 ports

3. **Curated templates are read-only.** Templates shipped with the AMI (in `cloudformation/`) should be integrity-checked. A SHA-256 manifest generated at build time, stored in the AMI, and verified before deployment.

4. **cfn-lint in CI.** Job 5 already validates templates with `cfn-lint`. Add custom rules that enforce the resource type allowlist and IAM policy constraints above.

---

## Threat Summary and Risk Register

| ID | Threat | Impact | Likelihood | Current Mitigation | Residual Risk | Action Required |
|----|--------|--------|------------|-------------------|---------------|-----------------|
| T1 | CFN `Resource: '*'` on instance role allows arbitrary stack deployment | Critical | Medium | None | **High** | Scope to `cj-*` stacks, add permission boundary |
| T2 | Malicious CFN template deploys admin IAM role | Critical | Low-Medium | cfn-lint syntax check only | **High** | Resource type allowlist, template content scanning |
| T3 | Prompt injection via assessment data manipulates Brain Agent | High | Medium | Structured JSON interface | Medium | Input sanitization, Controller validates tool calls, invocation limits |
| T4 | Lambda `ClientError` strings leak account info | Medium | High (already observed) | Generic 500 for unknown errors | Medium | Sanitize known error responses in check functions |
| T5 | Recipe tampering via S3 write access | High | Low | S3 versioning, KMS, TLS-only | Low-Medium | Object Lock for curated templates, write-permission scoping |
| T6 | Cross-account pivot via compromised instance | High | Low | ExternalId on cross-account role | Low | ExternalId stored in SSM; if SSM is readable by attacker, ExternalId is compromised. Consider AWS Secrets Manager instead. |
| T7 | Dashboard has no application-level authentication | Medium | Medium | Security group CIDR restriction | Medium | Add authentication mechanism before multi-user support |
| T8 | NiceGUI dependency tree is large and under-audited | Medium | Low | pip-audit in CI | Low-Medium | Pin all transitive dependencies, review pip-audit output actively |
| T9 | Brain Agent enters infinite tool-call loop | Low | Low | Lambda concurrency limits | Low | Add per-session invocation counter in Controller |
| T10 | Docker installed but unused increases attack surface | Low | Low | None | Low | Remove Docker from AMI if not needed |
| T11 | Bedrock permissions allow any model invocation | Low | Low | VPC endpoint | Low | Scope to specific model ARN(s) |
| T12 | Self-signed TLS cert on first boot | Low | Low | Documentation instructs replacement | Low | Consider Let's Encrypt or ACM integration |

---

## Gate Questions Answered

**What is the worst thing an adversary can do at each trust boundary?**

- User to Dashboard: Gain full control of CloudJanitor operations if network ACL is misconfigured. No application-layer auth blocks them.
- Brain Agent to Lambdas: Deploy any recipe that exists in S3. Cannot escape the three-Lambda tool boundary.
- Lambdas to AWS APIs: With current permissions, create arbitrary CloudFormation stacks in the account (T1, T2). This is the highest-impact boundary.
- S3 recipes: Replace a curated recipe with a malicious one, causing the next deployment to compromise the account (T5).

**If the IAM execution role is compromised, what is the blast radius?**

The EC2 instance role can create CloudFormation stacks with IAM capabilities on any resource in the account. Combined with STS cross-account assumption, an attacker could pivot to all accounts with the cross-account role deployed. This is the single largest risk and must be mitigated before production deployment.

**Does the IaC have the same threat coverage as the application code?**

No. The application code has schema validation, structured error handling, and input validation. The CloudFormation templates have syntax linting only. There is no semantic analysis of what the templates will deploy. A syntactically valid template that creates an admin backdoor role would pass all current checks. This gap must be closed with template content scanning and a resource type allowlist.
