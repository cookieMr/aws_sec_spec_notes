# AWS CloudFormation

<figure>
  <img src="./images/Arch_AWS-CloudFormation_64@5x.png" alt="AWS CloudFormation Icon" width=200>
  <figcaption><center>AWS CloudFormation<br><i>Image source: AWS Documentation</i></center></figcaption>
</figure>

**Overview**: AWS CloudFormation is an Infrastructure as Code (IaC) service that lets you define and provision AWS resources using templates (JSON or YAML). On the Security Specialty exam, CloudFormation is tested in the context of **deploying security baselines** (via StackSets), **secure template design** (IAM roles, encryption, VPC), **dynamic references for secrets**, and **policy-as-code with CloudFormation Guard**.

**Domain weight**: Appears across Management & Governance and Data Protection domains. CloudFormation is an **enabler for security automation** — typically 1–2 questions focused on secure deployment patterns.

---

## 1. Core Security Concepts

### 1.1. Stack Policies

- A **stack policy** defines what update actions are allowed or denied on specific resources
- Protects critical resources (e.g., database, security groups, IAM roles) from accidental updates or deletion
- When set, any stack update that would modify a protected resource is **denied** unless explicitly allowed
- Policy format: JSON with `Effect`, `Action`, `Principal`, `Resource`

```json
{
  "Statement": [
    {
      "Effect": "Deny",
      "Action": ["Update:Replace", "Update:Delete"],
      "Principal": "*",
      "Resource": "LogicalResourceId/MyDatabase"
    }
  ]
}
```

**Exam scenario**: A production database managed by CloudFormation must not be accidentally replaced or deleted during stack updates → apply a **stack policy** that denies `Update:Replace` and `Update:Delete` on the database resource.

### 1.2. Termination Protection

- When enabled, prevents a stack from being **deleted** via the CloudFormation console or API
- To delete, you must first disable termination protection
- Protects against accidental deletion of security-critical infrastructure
- Works independently of stack policies

**Exam scenario**: A security team wants to prevent accidental deletion of a critical logging stack → enable **termination protection** on the stack.

### 1.3. IAM Roles for CloudFormation

| IAM Role Type                     | Purpose                                                                      | Trusts                         |
| --------------------------------- | ---------------------------------------------------------------------------- | ------------------------------ |
| **CloudFormation execution role** | CloudFormation uses this role to create/modify/delete resources in the stack | `cloudformation.amazonaws.com` |
| **Service role**                  | Allows CloudFormation to perform operations on your behalf                   | `cloudformation.amazonaws.com` |

- **Execution role**: Attached to the stack — CloudFormation assumes this role when making API calls
- Without an execution role, CloudFormation uses the **user's permissions** (the user creating the stack)
- Security best practice: use a dedicated execution role with **least-privilege** permissions for the stack resources

```json
{
  "Effect": "Allow",
  "Action": ["ec2:*", "iam:PassRole", "s3:GetObject"],
  "Resource": "*"
}
```

Trust policy:

```json
{
  "Effect": "Allow",
  "Principal": {
    "Service": "cloudformation.amazonaws.com"
  },
  "Action": "sts:AssumeRole"
}
```

**Exam scenario**: A security requirement mandates that CloudFormation should use a dedicated IAM role with limited permissions when deploying resources, rather than using the user's permissions → create an **execution role** with least-privilege permissions and associate it with the stack.

### 1.4. IAM PassRole Permissions

- `iam:PassRole` is required when CloudFormation creates resources that need IAM roles (EC2, Lambda, ECS, etc.)
- The user creating the stack (or the execution role) must have `iam:PassRole` for the role ARN being passed
- **Critical security control**: without this, anyone who can create a stack could launch EC2 with an admin role

```json
{
  "Effect": "Allow",
  "Action": "iam:PassRole",
  "Resource": "arn:aws:iam::*:role/MyAppRole",
  "Condition": {
    "StringEquals": {
      "iam:PassedToService": ["ec2.amazonaws.com", "lambda.amazonaws.com"]
    }
  }
}
```

**Exam scenario**: A CloudFormation template creates an EC2 instance with an IAM instance profile → the CloudFormation execution role needs `iam:PassRole` permission for the instance profile role.

## 2. Dynamic References for Secrets

### 2.1. Overview

- CloudFormation supports **dynamic references** to retrieve values from:
  - **SSM Parameter Store**: `{{resolve:ssm:/path/to/parameter}}` and `{{resolve:ssm-secure:/path/to/parameter}}`
  - **Secrets Manager**: `{{resolve:secretsmanager:secret-id:secret-key}}`
- Values are resolved at **stack creation/update time**, not during template authoring
- Secrets are not stored in the template or in CloudFormation state — they are referenced at runtime

### 2.2. Secrets Manager Dynamic Reference

```yaml
Resources:
  MyRDSInstance:
    Type: AWS::RDS::DBInstance
    Properties:
      MasterUsername: "{{resolve:secretsmanager:prod/db/credentials:SecretString:username}}"
      MasterUserPassword: "{{resolve:secretsmanager:prod/db/credentials:SecretString:password}}"
```

- Format: `{{resolve:secretsmanager:<secret-id>:<secret-string-key>:<json-key>}}`
- `<secret-id>`: ARN, name, or name with version
- `<json-key>`: The key in the secret JSON (e.g., `username`, `password`)
- Works with secrets in the same or different account (with proper permissions)

### 2.3. SSM Parameter Store Dynamic Reference

```yaml
Resources:
  MyAppConfig:
    Type: Custom::Config
    Properties:
      ApiUrl: "{{resolve:ssm:/prod/app/api-url}}"
      ApiKey: "{{resolve:ssm-secure:/prod/app/api-key}}"
```

- `{{resolve:ssm:...}}` — resolves a plaintext parameter
- `{{resolve:ssm-secure:...}}` — resolves a SecureString parameter (decrypts via KMS)
- Format: `{{resolve:ssm-secure:<parameter-name>:<version>}}`

**Exam scenario**: A CloudFormation template needs to reference a database password stored in Secrets Manager without exposing it in the template → use `{{resolve:secretsmanager:...}}` dynamic reference.

## 3. CloudFormation StackSets

### 3.1. Purpose

- Deploy CloudFormation stacks across **multiple accounts** and **multiple regions**
- Critical for **security baselines**: deploy the same security controls (Config rules, GuardDuty, CloudTrail, SCPs, etc.) across all accounts in an AWS Organization

### 3.2. Key Features

| Feature                   | Description                                                           |
| ------------------------- | --------------------------------------------------------------------- |
| **Target accounts**       | All accounts in an Organization, specific OUs, or individual accounts |
| **Target regions**        | All regions or specific regions                                       |
| **Operation preferences** | Failure tolerance, max concurrent accounts, region order              |
| **Self-service vs admin** | Service-managed (Org auto-approval) vs self-managed (manual)          |
| **Automatic deployment**  | Automatically deploys to new accounts added to the Organization       |

### 3.3. Architecture

```
Admin Account (Security)
  ├── StackSet (CloudTrail baseline)
  │   ├── Account A - us-east-1
  │   ├── Account A - eu-west-1
  │   ├── Account B - us-east-1
  │   └── Account B - eu-west-1
  └── StackSet (Config rules baseline)
      ├── Account A - us-east-1
      └── Account B - us-east-1
```

### 3.4. Service-Managed vs Self-Managed StackSets

| Aspect                | Service-Managed                                         | Self-Managed                                |
| --------------------- | ------------------------------------------------------- | ------------------------------------------- |
| **Account targeting** | Based on AWS Organizations (OUs automatically detected) | Manually specify account IDs                |
| **New accounts**      | Automatically deployed to new accounts in the OU        | Must manually add new accounts              |
| **Trust**             | Uses Organization service-linked role                   | Requires manual trust setup across accounts |
| **Best for**          | Organizations with many accounts                        | Smaller orgs or non-Organization setups     |

**Exam scenario**: A security team needs to deploy CloudTrail, AWS Config, and GuardDuty across all accounts in an AWS Organization → use **CloudFormation StackSets** with service-managed deployment to automatically deploy to all current and future accounts.

## 4. CloudFormation Guard (Policy-as-Code)

### 4.1. Overview

- **CloudFormation Guard (cfn-guard)** is a policy-as-code tool for validating CloudFormation templates
- Ensures templates comply with security best practices **before deployment**
- Rule sets are written in a **Guard domain-specific language (DSL)**
- Can be enforced in CI/CD pipelines (CodePipeline, Jenkins, etc.)

### 4.2. Guard Rule Example

```
# Require S3 buckets to have encryption enabled
let s3_buckets = Resources.*[ Type == 'AWS::S3::Bucket' ]

rule require_s3_encryption when %s3_buckets !empty {
  %s3_buckets.Properties.BucketEncryption.ServerSideEncryptionConfiguration[0].ServerSideEncryptionByDefault.SSEAlgorithm in ['AES256', 'aws:kms']
  <<S3 Buckets must have server-side encryption enabled>>
}
```

### 4.3. Common Guard Rules for Security

| Rule                            | Purpose                                        |
| ------------------------------- | ---------------------------------------------- |
| S3 bucket encryption enabled    | Require SSE-S3 or SSE-KMS                      |
| S3 bucket public access blocked | Deny public ACLs and policies                  |
| EBS volumes encrypted           | Require encryption for all EBS volumes         |
| IAM role least privilege        | Limit `Action: "*"` and `Resource: "*"` usage  |
| Security group rules restricted | No ingress from `0.0.0.0/0` on sensitive ports |
| VPC flow logs enabled           | Require VPC Flow Logs for all VPCs             |
| CloudTrail enabled              | Require CloudTrail in all regions              |
| KMS key rotation enabled        | Require automatic key rotation                 |
| Encryption at rest required     | Check all supported resources have encryption  |

**Exam scenario**: A security team wants to prevent insecure CloudFormation templates from being deployed (e.g., S3 buckets without encryption) → use **CloudFormation Guard** as a policy-as-code validation step in the CI/CD pipeline.

## 5. Change Sets

### 5.1. Purpose

- Preview changes to a stack **before** applying them
- Shows: which resources will be added, modified, or deleted
- Indicates: `Modify`, `Replace` (resource recreated), `Delete`, `Add`
- Critical for security review of infrastructure changes

### 5.2. Security Implications

| Change Type | Risk Level | Description                                            |
| ----------- | ---------- | ------------------------------------------------------ |
| **Modify**  | Medium     | Resource properties are updated; data may be at risk   |
| **Replace** | High       | Resource is deleted and recreated — data loss possible |
| **Delete**  | Critical   | Resource is permanently deleted                        |

**Exam scenario**: A developer submits a CloudFormation template change for a production stack. The security team must review the changes before deployment → use **Change Sets** to preview the changes and approve or reject them.

## 6. Drift Detection

### 6.1. Purpose

- Detect when resources in a stack have been **manually modified** outside of CloudFormation
- Compares the **actual resource state** with the **template state**
- Reports: `IN_SYNC`, `DRIFTED`, `DELETED`, `NOT_CHECKED`
- Detects drift for many resource types (not all)

### 6.2. Security Implications

- Manual changes can introduce security vulnerabilities (e.g., someone opens a security group port manually)
- Drift detection alerts the security team to unauthorized changes
- Once detected, the team can **remediate** by rolling back to the template or updating the template to match

**Exam scenario**: A security team needs to detect when infrastructure is manually modified outside of CloudFormation (e.g., a security group rule is added directly via the console) → enable **drift detection** on CloudFormation stacks and monitor for `DRIFTED` status.

## 7. Deletion Policy

### 7.1. Overview

- `DeletionPolicy` attribute controls what happens to a resource when its stack is deleted
- Three values:

| Policy             | Behavior                                                       | Use Case                                         |
| ------------------ | -------------------------------------------------------------- | ------------------------------------------------ |
| `Delete` (default) | The resource is deleted when the stack is deleted              | Non-critical, recreatable resources              |
| `Retain`           | The resource is **preserved** when the stack is deleted        | Database data, critical logs, encryption keys    |
| `Snapshot`         | A snapshot is taken before deletion (supported resources only) | EBS volumes, RDS databases, ElastiCache clusters |

### 7.2. Security Implications

- `DeletionPolicy: Retain` is critical for security resources (e.g., CloudTrail logs, Config snapshots, S3 buckets with audit data)
- `DeletionPolicy: Snapshot` is important for forensic evidence (EBS snapshots, RDS snapshots)
- Without a `DeletionPolicy`, the resource is deleted — data loss

**Exam scenario**: A CloudFormation stack includes an S3 bucket with security audit logs. If the stack is deleted, the logs must not be lost → set `DeletionPolicy: Retain` on the S3 bucket resource.

## 8. CreationPolicy and UpdatePolicy

### 8.1. CreationPolicy

- Defines how CloudFormation waits for resource creation to complete
- `ResourceSignal`: Wait for a signal (e.g., from EC2 user-data script) before proceeding
- `AutoScalingCreationPolicy`: Wait for instances to be healthy in an ASG
- Security use case: Ensure EC2 instances complete bootstrapping and security agent installation before being considered "created"

### 8.2. UpdatePolicy

- Defines how CloudFormation handles updates to specific resource types
- `AutoScalingRollingUpdate`: How to update an ASG (batch size, pause time)
- `AutoScalingReplacingUpdate`: Replace instances during update
- Security use case: Control the pace of updates to ensure security agents are installed on new instances

## 9. CloudFormation Macros and Hooks

### 9.1. Macros

- **Macros** perform custom processing on templates at stack creation/update
- Supported use: transform templates, validate policies, inject parameters
- Example: AWS SAM is a macro (`Transform: AWS::Serverless-2016-10-31`)
- Can be used to **inject security defaults** (e.g., auto-add encryption to all resources)

### 9.2. Hooks (CloudFormation Hooks)

- **Hooks** are a newer feature for pre-deployment validation
- Run custom logic (via Lambda) **before** CloudFormation creates or updates a resource
- Can **fail** the deployment if the resource does not meet security requirements
- More integrated than Guard (Hooks are part of CloudFormation, Guard is a CLI tool)
- Enforce security policies at the **resource level** (e.g., "All S3 buckets must have encryption")

## 10. Security Best Practices

| Practice                                                | Description                                                                       |
| ------------------------------------------------------- | --------------------------------------------------------------------------------- |
| **Use execution roles**                                 | Dedicated IAM role for CloudFormation with least-privilege                        |
| **Use dynamic references for secrets**                  | Never hardcode secrets in templates; use `resolve:ssm` / `resolve:secretsmanager` |
| **Use StackSets for multi-account baselines**           | Deploy security controls consistently across all accounts                         |
| **Use CloudFormation Guard**                            | Validate templates against security policies before deployment                    |
| **Use Change Sets for review**                          | Preview changes before applying them to production                                |
| **Enable drift detection**                              | Detect unauthorized manual changes to infrastructure                              |
| **Use stack policies**                                  | Protect critical resources from accidental updates/deletion                       |
| **Enable termination protection**                       | Prevent accidental stack deletion                                                 |
| **Use `DeletionPolicy: Retain` for critical resources** | Protect audit logs, data, and encryption keys from deletion                       |
| **Use `iam:PassRole` restrictions**                     | Limit which roles CloudFormation can pass to resources                            |
| **Validate templates in CI/CD**                         | Run cfn-guard and linter checks before each deployment                            |
| **Use Hooks for pre-deployment validation**             | Enforce security policies at the resource level during deployment                 |
| **Use AWS::Include for reusable security templates**    | Create modular, reusable security template snippets                               |
| **Tag stacks for access control**                       | Use `aws:ResourceTag` conditions to control stack permissions                     |

## 11. Common Exam Scenarios

1. **Multi-account security baseline**: Deploy CloudTrail, Config, and GuardDuty to all accounts in an Organization → **CloudFormation StackSets** with service-managed deployment.

2. **Secret management in templates**: A template needs a database password → use **dynamic reference** `{{resolve:secretsmanager:...}}` (do NOT hardcode).

3. **Preventing accidental resource deletion**: A CloudFormation stack manages an S3 bucket with audit logs → set **`DeletionPolicy: Retain`** and **termination protection**.

4. **Template validation before deployment**: A CI/CD pipeline must reject templates that violate security policies → use **CloudFormation Guard** (cfn-guard) in the pipeline.

5. **Change review for production stacks**: Changes to production stacks must be reviewed → generate a **Change Set** and send it for approval.

6. **Execution role vs user permissions**: A user with admin permissions creates a stack but wants CloudFormation to use a restricted role → create an **execution role** with least-privilege permissions.

7. **Manual change detection**: Someone modified a security group directly via the console, causing drift → use **drift detection** to detect the unauthorized change.

8. **IAM PassRole for resource creation**: A CloudFormation template creates an EC2 instance → the execution role (or user) needs `iam:PassRole` for the instance profile role.

9. **Stack policy for production resources**: A production database must not be replaced during stack updates → apply a **stack policy** that denies `Update:Replace` on the database resource.

10. **Cross-account stack deployments**: Deploy security infrastructure to multiple member accounts from a central security account → use **CloudFormation StackSets**.

## 12. Exam Tips

1. **StackSets = multi-account deployment**: The answer for deploying security baselines across an AWS Organization.

2. **Dynamic references = secrets without hardcoding**: Use `resolve:secretsmanager` or `resolve:ssm-secure` — never hardcode secrets in templates.

3. **CloudFormation Guard = policy-as-code**: Validate templates before deployment. The exam might ask which tool enforces security policies on templates.

4. **Change Sets = preview before apply**: Essential for reviewing security-impacting changes.

5. **Drift detection = detect manual changes**: Monitor for unauthorized infrastructure changes outside CloudFormation.

6. **Stack policy ≠ termination protection**: Stack policy controls updates (what can be modified). Termination protection controls deletion.

7. **DeletionPolicy: Retain**: Critical for preserving security data (logs, backups, encryption keys) when the stack is deleted.

8. **Execution role**: CloudFormation uses this role to create/modify resources. Without it, the user's permissions are used.

9. **iam:PassRole**: Required when CloudFormation passes roles to resources (EC2, Lambda). Must be explicitly granted.

10. **Hooks vs Guard**: Guard validates the template file offline. Hooks run at deployment time — can inspect the actual state of resources.

11. **Service-managed StackSets**: Automatically deploys to new accounts added to an Organization — best for security baselines.

12. **StackSet operation preferences**: `FailureTolerance` (how many accounts can fail), `MaxConcurrentCount` (how many accounts to update simultaneously).

13. **CloudFormation template size**: Max 1 MB (template body) or 51 KB (template URL in S3). For larger templates, upload to S3 and reference the URL.

14. **CloudFormation supports YAML and JSON**: YAML is preferred for readability. Both support the same features.

15. **Transform section**: `AWS::Include` (include snippets from S3), `AWS::Serverless` (SAM), `AWS::CodeDeployBlueGreen` (blue/green deployments).

16. **CloudFormation resource ARN format**: Use `!GetAtt` or `!Ref` to reference resource attributes and IDs.

17. **Nested stacks**: Use `AWS::CloudFormation::Stack` to compose stacks from reusable templates. Good for modular security infrastructure.

18. **StackSets in the exam**: If the question involves deploying the same resources across multiple accounts, StackSets is almost always the answer.

19. **Changes are tracked in CloudTrail**: All CloudFormation API calls are logged — `CreateStack`, `UpdateStack`, `DeleteStack`, `CreateChangeSet`.

20. **CloudFormation is regional**: Stacks are regional resources. Use StackSets for multi-region deployments.
