# Phase 1 -- Problem Definition

**Project:** CloudJanitor -- AI-Powered AWS Security Configuration Agent

---

## The Problem

AWS accounts are created with almost no security controls enabled by default. CloudTrail is off. GuardDuty is off. AWS Config is not recording. There is no password policy beyond AWS minimums. Default encryption is disabled for EBS and S3. VPC Flow Logs do not exist. Security Hub is not aggregating findings. IAM Access Analyzer is not watching for external access.

Organizations with limited security expertise -- startups scaling past their first AWS account, mid-market companies without a dedicated cloud security team, agencies managing client environments -- inherit this default-insecure posture and do not know what they are missing. They pass weeks or months operating in production with no visibility into what is happening in their account.

Manual remediation requires a practitioner who understands dozens of AWS services, their interdependencies, their regional behavior, and the compliance framework mappings that justify each control. This person must then translate that knowledge into reproducible infrastructure-as-code, deploy it without breaking existing workloads, and verify that the controls are actually working. For a single account, this is a week of focused work. For an organization with ten accounts, it does not scale.

The gap is not knowledge availability -- AWS documentation is public and thorough. The gap is the operational capacity to translate that knowledge into deployed, verified, compliant infrastructure within a timeframe that matters.

---

## Problem Statement

AWS accounts ship insecure by default, and organizations without dedicated cloud security expertise lack the operational capacity to assess their security posture, deploy the right controls, and verify compliance -- leaving accounts exposed for weeks or months while manual remediation stalls on knowledge gaps and resource constraints.

---

## Gate Questions

### 1. What breaks in the real world if this is not built?

Organizations continue operating AWS accounts with no logging, no threat detection, no configuration tracking, and no compliance baseline. Specific real-world consequences:

- **No CloudTrail:** If an IAM credential is compromised, there is no record of what the attacker did. Incident response is impossible -- you cannot investigate what you did not log. The organization discovers the breach from their AWS bill, not from an alert.

- **No GuardDuty:** Cryptocurrency mining on hijacked EC2 instances runs undetected. Unauthorized API calls from foreign IP addresses generate no alerts. Data exfiltration through DNS tunneling is invisible.

- **No AWS Config:** Configuration drift is undetectable. Someone opens a security group to 0.0.0.0/0 and no one knows until it is exploited. There is no change history to reconstruct what happened.

- **No password policy:** Console users operate with 8-character passwords, no MFA requirement, no rotation. A single phished credential gives full console access.

- **No encryption defaults:** EBS volumes and S3 buckets are created unencrypted by default. Data at rest is exposed to anyone with physical access to the underlying storage or with read permissions to the resource.

- **No compliance posture:** The organization cannot answer basic audit questions. SOC2, ISO 27001, and CIS benchmarks all require these controls. Compliance failures delay sales deals, block partnerships, and create legal liability.

These are not hypothetical risks. AWS publishes case studies of exactly these failures. The 2019 Capital One breach exploited a misconfigured WAF and exfiltrated data through an instance role -- controls that a basic security baseline would have detected or prevented.

### 2. Why is code the right solution and not a process change, a configuration, or an existing tool?

**Why not a process change?** The process already exists -- AWS publishes CIS Benchmarks, SOC2 mapping guides, and Well-Architected Framework reviews. The bottleneck is not knowing what to do; it is having the capacity to do it. A checklist does not deploy CloudTrail. A playbook does not configure fourteen CloudWatch metric filters. Process changes work when the problem is human decision-making. This problem is operational throughput.

**Why not manual configuration through the AWS Console?** Console clicks are not reproducible, not auditable, and not idempotent. If you enable GuardDuty manually in us-east-1 and forget eu-west-1, you have a gap you may never discover. If you configure a password policy and someone changes it next month, there is no drift detection. Manual configuration is the current state of the problem, not the solution.

**Why not an existing tool?** The existing tool landscape breaks into categories:

- **AWS-native services (Security Hub, Control Tower, Organizations):** These are the building blocks, not the solution. Security Hub aggregates findings but does not deploy controls. Control Tower applies guardrails but requires AWS Organizations with specific constraints and is designed for new account setup, not remediating existing accounts. These services are what CloudJanitor deploys and configures -- they are the output, not a competitor.

- **CSPM tools (Prowler, ScoutSuite, CloudSploit):** These scan and report. They produce a list of findings. They do not remediate. The customer still needs someone to translate 200 findings into deployed infrastructure. Prowler tells you CloudTrail is off; it does not turn it on.

- **Enterprise platforms (Prisma Cloud, Wiz, Lacework):** These cost $50K-500K/year, require dedicated security teams to operate, and are designed for large enterprises. A 50-person company with three AWS accounts cannot justify this spend or staffing. CloudJanitor targets the market segment below these platforms.

- **IaC template libraries (CIS Benchmark CFN, AWS Samples):** These are raw CloudFormation templates without assessment, without reasoning about what the specific account needs, and without compliance verification after deployment. They are a step above manual configuration but still require expertise to select, customize, and validate.

Code is the right solution because the problem is fundamentally an automation gap -- the knowledge exists, the AWS APIs exist, the compliance mappings exist, but no tool in the current market combines assessment, intelligent recommendation, deployment, and verification into a single workflow that a non-expert can operate.

The AI component (Brain Agent) is justified specifically because recipe selection requires contextual reasoning. A greenfield account needs the SOC2 baseline. An account already running GuardDuty but missing Config needs a different subset. An account targeting HIPAA needs encryption controls prioritized. This selection logic is what separates CloudJanitor from a static template library.

---

## Output

The problem is clear. The real-world consequences are specific and verifiable. Code is the right solution because the bottleneck is operational capacity, not knowledge or process. Existing tools either scan-without-remediating, require enterprise budgets, or provide raw templates without assessment or verification. Proceed to Phase 2.
