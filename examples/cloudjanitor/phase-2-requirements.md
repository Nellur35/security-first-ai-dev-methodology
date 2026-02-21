# Phase 2 -- Product Requirements

**Project:** CloudJanitor -- AI-Powered AWS Security Configuration Agent
**Input:** Phase 1 Problem Statement
**Distribution:** AWS Marketplace AMI

---

## Functional Requirements

### FR-1: Assess Environment

The system must perform a read-only scan of an AWS account's security posture and produce a structured assessment report.

- Scan must cover: CloudTrail, GuardDuty, Security Hub, AWS Config, IAM password policy, IAM credential hygiene (access key age, MFA status, unused credentials), VPC Flow Logs, S3 public access block, EBS default encryption, KMS key rotation.
- Scan must operate across all active regions for multi-region services (CloudTrail, GuardDuty, Config, VPC Flow Logs).
- Output must be a structured JSON report with a per-service status (enabled/disabled/misconfigured), specific finding details, and a compliance gap summary.
- Scan must use only read/describe/list/get API calls. No write operations under any circumstances.
- The assessment Lambda must complete within 5 minutes for a typical account (fewer than 50 IAM users, fewer than 10 VPCs).

### FR-2: Deploy Security Recipes

The system must deploy security configurations through declarative recipes that map to CloudFormation templates.

- Each recipe is a YAML file conforming to a published schema. The schema defines: service configurations, compliance framework mappings, cost estimates, prerequisites, scope, and post-deployment validation checks.
- Five core recipes must ship in v1: SOC2 Compliance Baseline, IAM Hardening, Monitoring Baseline, Encryption Baseline, CIS Level 1.
- Each recipe maps to exactly one CloudFormation template. Deployment means creating or updating a CloudFormation stack.
- All resources created by a recipe must be tagged: `ManagedBy: CloudJanitor`, `RecipeId: <id>`, `RecipeVersion: <version>`.
- Deployment must be idempotent. Running the same recipe twice on the same account must result in a stack update (no-op if nothing changed), not a failure or duplicate resources.

### FR-3: Check Compliance

The system must verify that deployed recipes are functioning correctly and the account meets the expected security posture.

- Compliance checks must validate every assertion defined in a recipe's `validation` block.
- Output must be a structured compliance report: per-check pass/fail, evidence (the actual API response that proves the finding), and remediation guidance for failures.
- Compliance checks must be read-only. No mutations during verification.

### FR-4: Brain Agent (AI-Driven Decisions)

The system must include an AI reasoning agent that orchestrates assessment, recipe selection, deployment, and verification through natural language interaction.

- The Brain Agent runs on the EC2 instance and uses Amazon Bedrock (Claude) for reasoning.
- The Brain Agent must use a structured reasoning pipeline (TAISE: Think, Assess, Implement, Summarize, Evolve) encoded in its system prompt.
- The Brain Agent has access to three Lambda tools: `assess_environment`, `deploy_recipe`, `check_compliance`. It invokes them through standard Lambda invoke.
- The Brain Agent must be able to: recommend recipes based on assessment results, explain why a recipe is recommended, deploy a recipe upon user confirmation, verify deployment via compliance checks, and explain findings in plain language.
- A Complexity Router must classify incoming requests into pipeline variants: Quick (direct recipe lookup), Standard (multi-recipe reasoning), Full (complex/multi-org scenarios), Builder (custom recipe creation -- v1 stretch).
- The Brain Agent system prompt is a trade secret. It must be encrypted in AWS Secrets Manager, loaded at runtime, and never written to disk or logged.

### FR-5: Dashboard

The system must provide a web-based dashboard for interacting with the Brain Agent and managing recipes.

- NiceGUI (Python) application served on HTTPS port 443.
- Chat interface for Brain Agent interaction with visible reasoning traces.
- Recipe browser showing available recipes with descriptions, cost estimates, and compliance mappings.
- Deployment history showing deployed stacks, their status, and last compliance check results.
- Dashboard must be accessible only from within the VPC or through a specified CIDR block. No public internet access.

---

## Non-Functional Requirements

### NFR-1: Read-Only Assessment

The `assess_environment` function must use exclusively read-only AWS API calls. The IAM policy for this function must contain only `Describe*`, `Get*`, `List*`, and `Generate*` actions. No `Create*`, `Put*`, `Update*`, `Delete*`, or `Modify*` actions. This is a hard security constraint -- a customer must be able to run assessment without granting write access.

### NFR-2: CloudFormation-Only Deployment

All resource provisioning must go through CloudFormation stacks. No direct API calls that create, modify, or delete AWS resources outside of CloudFormation. This ensures: full auditability (CloudFormation events log every change), rollback capability (failed stacks roll back automatically), drift detection (CloudFormation drift detection shows post-deployment changes), and clean removal (deleting the stack removes all resources).

### NFR-3: Idempotent Operations

Every operation must be safe to retry. Deploying the same recipe twice must result in a CloudFormation stack update, not an error. Running assessment twice must produce consistent results. Running compliance checks twice must not create side effects.

### NFR-4: Sub-5-Minute Assessment

A full environment assessment must complete in under 5 minutes for a standard account (fewer than 50 IAM users, fewer than 10 VPCs, fewer than 5 active regions). This is a usability constraint -- a user who clicks "assess" and waits 20 minutes will not trust the tool.

### NFR-5: Zero Internet Egress

The EC2 instance must operate without an internet gateway or NAT gateway. All AWS API calls must route through VPC endpoints (Interface endpoints for Bedrock, Lambda, CloudWatch Logs, SSM, STS; Gateway endpoint for S3). This eliminates an entire class of data exfiltration vectors and satisfies network segmentation requirements for SOC2 and CIS benchmarks.

### NFR-6: CIS Level 2 Hardened AMI

The base AMI must be hardened to CIS Amazon Linux 2023 Benchmark Level 2. This includes: filesystem hardening, service minimization, audit logging, network configuration, and access control. The AMI build process (Packer) must include a hardening script that is auditable and reproducible.

### NFR-7: Encryption at Rest and in Transit

All data at rest must be encrypted. S3 buckets use SSE-KMS. EBS volumes use encrypted gp3. Secrets Manager encrypts the Brain Agent system prompt. All data in transit uses TLS (HTTPS for dashboard, TLS for VPC endpoint communication, TLS for Bedrock API calls).

### NFR-8: Structured Logging

All components must use structured JSON logging with correlation IDs. Logs must never contain secrets, credentials, full API responses with sensitive data, or the Brain Agent system prompt. Log level must be configurable via SSM Parameter Store without restarting the service.

---

## Explicit Exclusions

These are deliberate scope boundaries, not items deferred due to time pressure. Each exclusion has a reason.

### CloudJanitor is NOT a SIEM

It does not collect, correlate, or store security events from the customer's environment. It deploys the services that do (CloudTrail, GuardDuty, Security Hub) but does not replicate their functionality. Customers who need a SIEM should use the services CloudJanitor deploys, or a dedicated platform.

### CloudJanitor is NOT real-time monitoring

It does not run continuous scans or watch for configuration changes in real-time. Assessment and compliance checks are on-demand or scheduled (via EventBridge). Real-time monitoring is the job of AWS Config Rules, GuardDuty, and Security Hub -- services that CloudJanitor deploys and configures.

### CloudJanitor does NOT support multi-account management in v1

The v1 deployment model is single-account: the AMI runs in one account and manages that account. Cross-account role assumption is architected (the `cross-account-role.yaml` template exists) but multi-account orchestration, StackSets integration, and Organizations-level management are v2 scope.

### CloudJanitor does NOT support multi-cloud

AWS only. No Azure, no GCP, no on-premises. The recipe schema, CloudFormation deployment model, and VPC endpoint architecture are all AWS-native by design. Multi-cloud would require a fundamentally different architecture.

### CloudJanitor does NOT build custom scanning engines

It uses AWS-native services for detection: GuardDuty for threat detection, Security Hub for finding aggregation, AWS Config for configuration compliance, IAM Access Analyzer for external access analysis. CloudJanitor does not replicate or replace these services with custom detection logic.

### CloudJanitor does NOT provide a community recipe marketplace in v1

The recipe YAML schema is intentionally open and extensible. A community marketplace for sharing recipes is a v2 feature. In v1, recipes are curated by CloudJanitor or created locally by the Brain Agent.

---

## Definition of Done

Done is not "the code works locally." Done is a customer with no AWS security expertise who can:

1. **Launch the AMI** from AWS Marketplace into their VPC using the provided CloudFormation template, filling in parameters (VPC ID, subnet, CIDR, email) without needing to understand the underlying architecture.

2. **Open the dashboard** at the instance's private IP on port 443 and see a functional chat interface.

3. **Ask the Brain Agent** "What is the security posture of my account?" and receive a structured assessment within 5 minutes that correctly identifies which security services are enabled and which are missing.

4. **Receive a recommendation** from the Brain Agent for which recipe to deploy based on the assessment results, with an explanation of why that recipe addresses the identified gaps.

5. **Confirm the deployment** and watch the Brain Agent deploy the recipe as a CloudFormation stack, with tagged resources that are visible in the CloudFormation console.

6. **Run a compliance check** and see a report confirming that the deployed services are active and correctly configured, or identifying specific failures with remediation guidance.

7. **Re-run the assessment** and see the security posture score improve, reflecting the controls that are now enabled.

8. **Delete the CloudFormation stack** and see all CloudJanitor-created resources cleaned up, leaving the account in its pre-deployment state (minus any services like CloudTrail that should remain enabled).

If any of these steps require the customer to read AWS documentation, SSH into the instance, or understand CloudFormation internals, the product is not done.

---

## Gate Questions

### Is every requirement testable?

Yes. Each functional requirement maps to specific Lambda invocations with verifiable outputs:

| Requirement | Test |
|-------------|------|
| FR-1 (Assess) | Invoke `assess_environment` against a test account, verify JSON schema of output, verify all services are checked, verify no write API calls in CloudTrail |
| FR-2 (Deploy) | Invoke `deploy_recipe` with SOC2 baseline, verify CloudFormation stack created, verify all resources tagged, invoke again and verify stack update (not create) |
| FR-3 (Compliance) | Deploy recipe, invoke `check_compliance`, verify each validation check returns pass/fail with evidence |
| FR-4 (Brain Agent) | Provide assessment input, verify Brain Agent selects correct recipe, verify it invokes correct Lambda tools in correct order |
| FR-5 (Dashboard) | HTTPS request to port 443, verify response, verify chat interface accepts input and returns Brain Agent response |

Non-functional requirements are testable through:

| Requirement | Test |
|-------------|------|
| NFR-1 (Read-only) | Parse IAM policy JSON, assert no write actions. Run assessment and check CloudTrail for write events -- must find zero |
| NFR-2 (CFN-only) | Grep all Lambda handler code for boto3 `create_*`, `delete_*`, `update_*`, `put_*` calls outside CloudFormation -- must find zero |
| NFR-3 (Idempotent) | Deploy same recipe twice, assert second invocation is a stack update not a new stack. Assert no errors |
| NFR-4 (Sub-5-min) | Time the `assess_environment` Lambda in a test account, assert under 300 seconds |
| NFR-5 (Zero egress) | Deploy instance without internet gateway or NAT, verify all operations succeed through VPC endpoints only |

### What is explicitly out of scope?

See Explicit Exclusions above. Summarized: not a SIEM, not real-time monitoring, not multi-account in v1, not multi-cloud, no custom scanning engines, no community marketplace in v1.

### What does done look like in reality, not in a CI dashboard?

A non-technical customer launches the AMI, talks to the Brain Agent in plain English, gets their AWS account assessed and hardened within 30 minutes, and can prove compliance to an auditor by showing the compliance report. See Definition of Done for the specific steps.

---

## Decisions & Rejected Alternatives

### CloudFormation over Terraform

**Decision:** All resource deployment uses CloudFormation.

**Rejected:** Terraform. Terraform requires the customer to install and manage Terraform state. AWS Marketplace AMIs cannot bundle HashiCorp's BSL-licensed binary without licensing complications. CloudFormation is native to every AWS account, requires no additional tooling, produces native CloudFormation events in CloudTrail, supports drift detection natively, and aligns with the zero-dependency deployment model. The tradeoff is that CloudFormation's DSL is more verbose and its update behavior is less predictable than Terraform's plan-and-apply model. This tradeoff is acceptable because CloudJanitor generates the templates -- the verbosity cost is paid by the developer, not the customer.

### CloudFormation stacks over direct API calls

**Decision:** Recipes deploy by creating CloudFormation stacks, not by making direct AWS API calls.

**Rejected:** Direct boto3 calls to enable services (e.g., `guardduty.create_detector()`). Direct calls are faster to implement and simpler to code. However, they produce no deployment artifact, no rollback capability, no drift detection, and no single point of removal. A customer who wants to undo a CloudJanitor deployment would need to manually identify and delete every resource. CloudFormation stacks provide all of this for free. The tradeoff is deployment latency (CloudFormation stacks take 2-5 minutes vs. seconds for direct calls) and debugging complexity (CloudFormation error messages are notoriously unhelpful). Both are acceptable for a tool that runs infrequently.

### AMI over SaaS multi-tenant

**Decision:** CloudJanitor ships as an AMI that runs in the customer's account.

**Rejected:** SaaS model where CloudJanitor runs in our account and assumes cross-account roles into customer accounts. The SaaS model requires customers to trust a third party with write access to their AWS security services -- a hard sell for security-conscious buyers. It also requires building multi-tenant infrastructure, authentication, billing integration, and SOC2 certification for our own environment before we can sell a security product. The AMI model means the customer's data never leaves their account, Bedrock API calls run on their bill, and the trust boundary is the IAM role they control. The tradeoff is that we lose visibility into usage telemetry and cannot push updates without the customer redeploying. Both are solvable in v2 with optional phone-home and AMI update mechanisms.

### Bedrock over self-hosted model

**Decision:** Brain Agent uses Amazon Bedrock for LLM inference.

**Rejected:** Bundling a self-hosted model (e.g., Llama, Mistral) on the EC2 instance. Self-hosting would require a much larger instance type (p3/g5 for GPU inference), dramatically increasing the AMI cost. Bedrock is available in the customer's account, charges per-token, and requires no GPU instances. The reasoning quality of Claude through Bedrock exceeds what a small self-hosted model can deliver for the recipe selection and explanation tasks. The tradeoff is a dependency on Bedrock availability and a per-token cost that the customer bears. This is acceptable because the token volumes for security assessment reasoning are low (a few thousand tokens per interaction).

### Structured YAML recipes over freeform LLM generation

**Decision:** Security configurations are defined as YAML recipes with a strict schema, validated by deterministic code (not LLM).

**Rejected:** Having the Brain Agent generate CloudFormation templates directly from natural language. LLM-generated infrastructure code is unpredictable -- a single hallucinated IAM permission or misconfigured security group could create the vulnerability the tool is supposed to prevent. The recipe schema constrains the output space to known-safe configurations. The Brain Agent selects and recommends recipes; it does not author arbitrary CloudFormation. The tradeoff is reduced flexibility -- the system can only deploy configurations that exist as recipes. This is a feature, not a limitation. For a security tool, constrained correctness is more valuable than unconstrained flexibility.

### Lambda tools over direct Bedrock tool-use

**Decision:** Brain Agent capabilities are implemented as Lambda functions invoked through standard `lambda:InvokeFunction`.

**Rejected:** Implementing tools as inline Python code executed by the Brain Agent process on the EC2 instance. Lambda provides: IAM-scoped execution (each function has only the permissions it needs), timeout enforcement (300 seconds max), isolated execution (a bug in assessment cannot crash the Brain Agent), CloudWatch logging (every invocation is logged independently), and independent deployability (update a Lambda without redeploying the AMI). The tradeoff is invocation latency (~100ms cold start) and the operational complexity of managing Lambda deployment. Both are negligible for a tool that runs assessments on-demand.

### VPC endpoints over NAT gateway

**Decision:** All AWS API access routes through VPC endpoints. No NAT gateway.

**Rejected:** NAT gateway for outbound internet access. A NAT gateway is simpler to configure and provides access to all AWS API endpoints automatically. However, it also provides a path to the public internet, which means: the instance could be used for data exfiltration if compromised, the instance could reach arbitrary external endpoints, and the network architecture fails the "zero egress" requirement that SOC2 CC6.6 and CIS benchmarks expect. VPC endpoints are more expensive (per-endpoint hourly charge) and require explicit configuration for each service, but they eliminate the internet egress path entirely. For a security product, the zero-egress posture is non-negotiable.
