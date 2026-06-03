# AWS Config

<figure>
  <img src="./images/Arch_AWS-Config_64@5x.png" alt="AWS Config Icon" width=200>
  <figcaption><center>AWS Config<br><i>Image source: AWS Documentation</i></center></figcaption>
</figure>

**Overview**: AWS Config is a service that evaluates your AWS resource configurations against desired policies. It records configuration changes over time, enabling security auditing, compliance monitoring, and change management. Config answers the questions "what resources do I have?", "what changed and when?", and "are my resources compliant with my policies?"

**Domain weight**: AWS Config is the most important service in the Management and Security Governance domain (~14% of SCS-C03) and is frequently tested alongside Security Hub, GuardDuty, and IAM for compliance and auditing scenarios.

## 1. Core Concepts

| Concept                    | Description                                                                 |
| -------------------------- | --------------------------------------------------------------------------- |
| **Configuration item**     | A snapshot of a resource's configuration at a point in time                 |
| **Configuration history**  | The sequence of configuration items for a resource over time (stored in S3) |
| **Configuration snapshot** | A point-in-time snapshot of all tracked resources (delivered to S3)         |
| **Configuration stream**   | A real-time stream of configuration changes delivered to SNS                |
| **Config rule**            | A policy that evaluates whether resources are compliant                     |
| **Conformance pack**       | A collection of Config rules and remediation actions packaged together      |
| **Aggregator**             | A central view of Config data across multiple accounts and regions          |

## 2. Configuration Recorder

### 2.1. What It Does

- Records configuration changes for supported AWS resources
- Stores the configuration history and timeline for each resource
- Can record **global resources** (IAM, CloudFront, WAF) — must be explicitly enabled

### 2.2. Recording Modes

| Mode           | Description                                       | Default  |
| -------------- | ------------------------------------------------- | -------- |
| **Continuous** | Records all configuration changes as they happen  | Default  |
| **Daily**      | Records configuration changes once every 24 hours | Optional |

- Continuous recording is recommended for security auditing
- The configuration recorder can be stopped (but this is a security concern — monitor with CloudTrail)

### 2.3. Resource Types

- Config supports 300+ AWS resource types
- By default, records **all supported resources** in the region
- Can be scoped to record only specific resource types
- Global resources (IAM, CloudFront, WAF, Route 53) require recording in **us-east-1**

**Exam scenario**: IAM changes are not appearing in AWS Config → the configuration recorder may not be recording **global resources**, or the recorder is not configured in **us-east-1**.

## 3. Config Rules

### 3.1. Managed Rules vs Custom Rules

| Type              | Source                                            | Examples                                                                                                                     |
| ----------------- | ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| **Managed rules** | AWS-provided, ready-to-use rules                  | `s3-bucket-public-read-prohibited`, `iam-user-mfa-enabled`, `cloud-trail-enabled`, `ec2-encrypted-volumes`, `restricted-ssh` |
| **Custom rules**  | Your own Lambda function that evaluates resources | Custom logic for your specific compliance requirements                                                                       |

### 3.2. Rule Evaluation

- **Trigger types**:
  - **Configuration changes**: The rule is evaluated when a relevant resource changes
  - **Periodic**: The rule is evaluated at a specified frequency (e.g., every 6 hours)
- **Compliance states**:

| State                 | Meaning                                       |
| --------------------- | --------------------------------------------- |
| **Compliant**         | The resource conforms to the rule             |
| **Non-compliant**     | The resource does not conform to the rule     |
| **Not applicable**    | The rule does not apply to this resource type |
| **Insufficient data** | Config lacks data to evaluate the resource    |

### 3.3. Common Exam Security Rules

| Managed Rule                               | What It Checks                               |
| ------------------------------------------ | -------------------------------------------- |
| `s3-bucket-public-read-prohibited`         | No public read ACLs on S3 buckets            |
| `s3-bucket-public-write-prohibited`        | No public write ACLs on S3 buckets           |
| `s3-bucket-ssl-requests-only`              | Enforce HTTPS on S3 bucket policies          |
| `s3-bucket-server-side-encryption-enabled` | SSE enabled on S3 buckets                    |
| `s3-bucket-logging-enabled`                | S3 access logging is enabled                 |
| `iam-user-mfa-enabled`                     | IAM users have MFA enabled                   |
| `iam-password-policy`                      | Password policy meets requirements           |
| `root-account-mfa-enabled`                 | Root user has MFA                            |
| `cloud-trail-enabled`                      | CloudTrail is enabled                        |
| `cloud-trail-encryption-enabled`           | CloudTrail logs are encrypted                |
| `cloud-trail-log-file-validation-enabled`  | CloudTrail integrity validation is on        |
| `ec2-encrypted-volumes`                    | EBS volumes are encrypted                    |
| `ec2-instances-in-vpc`                     | EC2 instances are in a VPC (not EC2-Classic) |
| `restricted-ssh`                           | Security groups block SSH from 0.0.0.0/0     |
| `ec2-ebs-snapshot-public-restorable`       | EBS snapshots are not public                 |
| `vpc-sg-open-only-to-authorized-ports`     | Security groups are restrictive              |

### 3.4. Custom Rules (Lambda)

- Custom rules evaluate resources using a Lambda function
- The Lambda function receives a configuration item and returns a compliance state
- Use case: Evaluate resource configurations that cannot be checked by managed rules
- **Exam scenario**: A specific compliance requirement cannot be met by any managed Config rule → create a **custom Config rule** using a Lambda function.

## 4. Conformance Packs

### 4.1. Purpose

- A collection of Config rules and remediation actions packaged as a single template (YAML)
- Deploy a complete compliance framework to one or more accounts/regions at once
- Based on AWS CloudFormation templates
- Pre-built packs available: **CIS AWS Foundations**, **PCI DSS**, **NIST 800-53**, **FSBP (Foundational Security Best Practices)**

### 4.2. Key Features

- Deploy rules + remediation across accounts and regions in a single operation
- Cannot be edited after creation — must be redeployed with changes
- Individual rules within a pack can be disabled
- Reports the compliance status of each rule in the pack

**Exam scenario**: A security team needs to deploy the same set of 20 Config rules across 50 accounts → create a **conformance pack** and deploy it to the target accounts/regions.

## 5. Remediation

### 5.1. Automatic Remediation

- Config rules can trigger **SSM Automation documents** as remediation actions
- When a resource is found non-compliant, the remediation runs automatically
- Example: An `s3-bucket-public-read-prohibited` rule detects a public bucket → SSM Automation runs a document that removes the public ACL

### 5.2. Manual Remediation

- Non-compliant resources are listed in the Config dashboard
- Administrators can manually investigate and fix
- Config provides details about what specifically is non-compliant

**Exam scenario**: A bucket is found non-compliant for public read access. The security team wants to automatically fix it → attach an **SSM Automation document** as a remediation action to the Config rule.

## 6. Aggregators

### 6.1. Purpose

- Provides a **single pane of glass** for Config data across multiple accounts and regions
- Aggregates configuration details, compliance status, and resource inventory
- Can aggregate from all accounts in AWS Organizations

### 6.2. Authorized Account Aggregation

- **Source accounts**: Individual accounts or organization member accounts
- **Aggregator account**: A central account (typically security/audit account) that collects data
- Use case: Central security team monitors compliance across the entire organization

**Exam scenario**: A security team needs to view the compliance status of all resources across 100 accounts and 15 regions in one place → use **AWS Config Aggregator** in the central security account.

## 7. Integration with Other Services

| Service             | Integration                                                                          |
| ------------------- | ------------------------------------------------------------------------------------ |
| **Security Hub**    | Config findings are forwarded to Security Hub for centralized security management    |
| **CloudTrail**      | CloudTrail logs all Config API calls (PutConfigRule, PutConfigurationRecorder, etc.) |
| **CloudWatch**      | Config sends compliance change notifications via SNS/EventBridge                     |
| **SNS**             | Configuration stream delivers real-time change notifications                         |
| **S3**              | Configuration history and snapshots are stored in S3                                 |
| **EventBridge**     | Compliance state changes trigger event-driven automation                             |
| **Systems Manager** | SSM Automation documents run remediation actions                                     |

## 8. Config Data Storage

| Data Type                   | Destination                                | Retention          |
| --------------------------- | ------------------------------------------ | ------------------ |
| **Configuration history**   | S3 bucket (configured by delivery channel) | Indefinite (S3)    |
| **Configuration snapshots** | S3 bucket                                  | Indefinite (S3)    |
| **Compliance history**      | S3 bucket                                  | Indefinite (S3)    |
| **Configuration stream**    | SNS topic                                  | Configurable (SNS) |
| **Advanced queries**        | Queryable directly in Config console       | N/A                |

### 8.1. Delivery Channel

- A delivery channel specifies the destination for Config data:
  - **S3 bucket** (required) — for history, snapshots, and compliance data
  - **SNS topic** (optional) — for real-time configuration stream notifications
- The S3 bucket must have a policy that grants Config write permissions
- **Exam scenario**: Config data is not being delivered → check the **delivery channel** configuration and S3 bucket policy.

## 9. Advanced Queries

### 9.1. Purpose

- Query the current configuration state of your resources using SQL-like queries
- No need to export data — queries run directly against the Config database
- Useful for resource discovery and inventory

### 9.2. Query Examples

```sql
SELECT
  resourceId,
  resourceType,
  configuration.instanceType
WHERE
  resourceType = 'AWS::EC2::Instance'
  AND configuration.instanceType LIKE 't2.%'
```

### 9.3. Use Cases

- Find all EC2 instances with a specific instance type
- Find all S3 buckets without encryption enabled
- Find all IAM roles with a specific policy attached
- Resource inventory for compliance reporting

## 10. Security Best Practices

### 10.1. Configuration

- **Enable Config in all regions and all accounts** — resource changes in any region can affect security
- **Record global resources** — IAM, CloudFront, and WAF changes are critical security events
- **Use continuous recording** (not daily)
- **Set up an aggregator** for centralized monitoring
- **Enable Config rules** for foundational security best practices:
  - S3 bucket public access prohibitions
  - CloudTrail enabled and encrypted
  - IAM user MFA enforced
  - EBS volume encryption
  - Security group restrictions

### 10.2. Monitoring

- Monitor Config compliance changes via **SNS** or **EventBridge**
- Forward non-compliant findings to **Security Hub**
- Use **conformance packs** to deploy compliance frameworks

### 10.3. Remediation

- Automate remediation of common non-compliant findings using **SSM Automation documents**
- Start with manual remediation for sensitive resources
- Test remediation actions in non-production environments first

## 11. Limits and Quotas

| Resource                                    | Limit          |
| ------------------------------------------- | -------------- |
| Config rules per region per account         | 500            |
| Conformance packs per region per account    | 50             |
| Rules per conformance pack                  | 200            |
| Max evaluation results per rule evaluation  | 100            |
| Max S3 bucket size for delivery             | No limit       |
| Resources covered                           | 300+ types     |
| Config rule evaluation frequency (periodic) | Minimum 1 hour |

## 12. AWS Config Cost Considerations

- Charged per **configuration item recorded** — each resource change generates a CI
- Charged per **Config rule evaluation** — each time a rule evaluates resources
- Conformance packs are charged based on the number of rules within them
- Cost can be significant in large environments with frequent changes
- Can be reduced by:
  - Recording only specific resource types (if not all are needed)
  - Using fewer periodic rules
  - Using configuration change-triggered rules instead of periodic rules where possible

## 13. Config vs Other Services

| Service          | What It Does                                                                 |
| ---------------- | ---------------------------------------------------------------------------- |
| **AWS Config**   | Tracks resource configuration changes and evaluates compliance against rules |
| **CloudTrail**   | Records API calls (who, what, when) — the "who" behind Config changes        |
| **GuardDuty**    | Detects threats and malicious activity                                       |
| **Security Hub** | Aggregates findings and compliance status from multiple services             |
| **Inspector**    | Scans for vulnerabilities (network, host, container)                         |

**Exam tip**: If the question asks about **configuration changes over time** or **compliance with policies**, the answer is AWS Config. If it asks about **API activity** (who made the change), the answer is CloudTrail. They are complementary — CloudTrail tells you who changed something, Config tells you what the state is now.

## 14. Exam Tips

1. **Config records resource configurations, CloudTrail records API calls**. Config tells you what a resource looks like now and what it looked like before. CloudTrail tells you who made the change.

2. **Config rules evaluate compliance**. Managed rules cover common benchmarks (S3 public access, CloudTrail enabled, MFA). Custom rules use Lambda for anything not covered by managed rules.

3. **Conformance packs** deploy a collection of rules and remediation as a single template. Use them to apply compliance frameworks (CIS, PCI DSS, NIST, FSBP) across accounts.

4. **Aggregators** provide a central view of Config data across accounts and regions. Essential for multi-account environments.

5. **Remediation** uses SSM Automation documents to automatically fix non-compliant resources. E.g., automatically remove public S3 access when detected.

6. **Delivery channel** = S3 bucket (required) + SNS topic (optional). Without a properly configured delivery channel, Config data is not stored.

7. **Global resources** (IAM, CloudFront, WAF) must be recorded in **us-east-1**. If you don't see IAM changes in Config, check that the recorder covers global resources.

8. **Continuous recording** is the default and recommended mode. Daily recording misses changes between snapshots.

9. **Configuration stream** (SNS) provides real-time notifications of resource changes.

10. **Advanced queries** let you run SQL-like queries against your current resource configurations without exporting data.

11. **Trigger types**: **Configuration change** rules evaluate on changes. **Periodic** rules evaluate at fixed intervals. Use change-triggered rules where possible for cost efficiency.

12. **Config + Security Hub**: Config findings are forwarded to Security Hub automatically when both are enabled.

13. **Config does not prevent changes** — it detects and reports them. Use SCPs or IAM policies to prevent, and Config to detect.

14. **300+ resource types** are supported — Config coverage is broad but not every AWS resource is tracked. Check documentation for supported types.

15. **Config rules are regional** — a rule created in us-east-1 does not apply to resources in eu-west-1. Use conformance packs to deploy rules across regions.
