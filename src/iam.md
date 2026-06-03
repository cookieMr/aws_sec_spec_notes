# AWS IAM

<figure>
  <img src="./images/iam.png" alt="AWS IAM Icon" width=200>
  <figcaption><center>AWS IAM<br><i>Image source: <a href="https://vecta.io/symbols/23/aws-security-identity-compliance/11/iam">Vecta.io</a></i></center></figcaption>
</figure>

**Overview**: IAM is the foundation of AWS security. It controls who can do what to which resources. The exam heavily tests layered policy questions — you must understand how identity-based policies, resource-based policies, permission boundaries, session policies, and SCPs interact.

**Domain weight**: ~20% of the SCS-C03 exam — the single largest domain.

## 1. Five Policy Types

There are five policy types that affect authorization in AWS. Understanding the distinction between each is critical for the exam.

| Policy Type                  | Attachment Point                               | Default Effect                       | Description                                                   |
| ---------------------------- | ---------------------------------------------- | ------------------------------------ | ------------------------------------------------------------- |
| Identity-based               | User, Group, Role                              | Allow                                | Grants permissions directly to a principal                    |
| Resource-based               | Resource (S3 bucket, KMS key, SQS queue, etc.) | Allow                                | Grants access from the resource side                          |
| Permissions Boundary         | User or Role                                   | Caps (allow must be within boundary) | Sets the maximum permissions a principal can ever have        |
| SCP (Service Control Policy) | OU, Account, Root                              | Allow (but can deny)                 | Organization-wide guardrails that cap all accounts under them |
| Session Policy               | STS AssumeRole/GetSessionToken                 | Restricts                            | Further narrows permissions for a temporary session           |

## 2. Policy Evaluation Logic

**The golden rule: Explicit Deny > Allow > Implicit Deny**

### 2.1. Evaluation Algorithm

1. If an SCP denies the action → explicit deny (cannot be overridden by any other policy)
2. Resource-based policies are evaluated for the requesting principal
3. Identity-based policies are evaluated for the requesting principal
4. Permissions boundary is checked — the action must be within the boundary
5. Session policy restricts the effective permissions further (if passed)
6. If any policy type has an explicit Deny → Deny
7. If there is an Allow and no explicit Deny → Allow
8. If there is no Allow → Implicit Deny (AccessDenied)

### 2.2. Key Exam Points

- **Explicit Deny always wins** — even overriding an Allow in the same policy statement
- **SCP Deny beats everything** — you cannot override an SCP deny with an identity-based Allow
- **SCP Allow does NOT grant permissions** — it only sets a maximum boundary. The principal still needs an identity-based Allow
- **Permission Boundaries** also cap without granting — the identity-based policy must grant the action AND the action must fall within the boundary
- **Session policies** are restrictive — if the identity policy grants `ec2:*` but the session policy only allows `ec2:Describe*`, the session gets only `ec2:Describe*`

## 3. Policy Structure

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::example-bucket/*",
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": "10.0.0.0/8"
        }
      }
    }
  ]
}
```

### 3.1. Statement Elements

| Element     | Required                | Description                                                     |
| ----------- | ----------------------- | --------------------------------------------------------------- |
| `Effect`    | Yes                     | Either `Allow` or `Deny`                                        |
| `Action`    | Yes                     | The API action or actions being permitted or denied             |
| `Resource`  | Yes\*                   | The ARN or ARNs of the resource(s) the policy applies to        |
| `Principal` | Only for resource-based | The entity the policy applies to (user, role, account, service) |
| `Condition` | No                      | Specifies when the policy is in effect based on context keys    |
| `Sid`       | No                      | Optional identifier for the statement, useful for debugging     |

\* `Resource` is required in identity-based policies but becomes optional when using `NotResource`.

### 3.2. Special Elements

- **`NotAction`** — matches every action EXCEPT the ones listed. Useful for blocking all actions except a few. For example, `"NotAction": "iam:*"` with a Deny effect would deny everything except IAM actions.
- **`NotResource`** — matches every resource EXCEPT the ones listed. The inverse of `Resource`.
- **`NotPrincipal`** — (resource-based policies only) matches every principal EXCEPT the ones specified. The inverse of `Principal`.

## 4. Condition Keys (Critical for Exam)

Conditions are where IAM policies get nuanced. The exam frequently tests scenario-based questions where a condition key is the deciding factor between an Allow and a Deny.

### 4.1. Global Condition Keys

These condition keys work across all AWS services:

| Condition Key                | Purpose                                                              | Type      |
| ---------------------------- | -------------------------------------------------------------------- | --------- |
| `aws:SourceIp`               | Restrict to specific IP address ranges                               | IpAddress |
| `aws:SourceVpc`              | Restrict to requests originating from a specific VPC                 | String    |
| `aws:SourceVpce`             | Restrict to requests from a specific VPC Endpoint                    | String    |
| `aws:SourceOrgID`            | Verify the request came from an account in your organization         | String    |
| `aws:SourceAccount`          | For cross-account access, verify the external account ID             | String    |
| `aws:MultiFactorAuthPresent` | Require MFA authentication (`Bool` true/false)                       | Bool      |
| `aws:CurrentTime`            | Time-based conditions (date/time format)                             | Date      |
| `aws:EpochTime`              | Time-based conditions (epoch timestamp)                              | Numeric   |
| `aws:RequestedRegion`        | Restrict which AWS regions can be used                               | String    |
| `aws:PrincipalType`          | The type of principal (User, AssumedRole, FederatedUser, Root, etc.) | String    |
| `aws:PrincipalOrgID`         | The organization ID that the principal's account belongs to          | String    |
| `aws:PrincipalTag`           | Match a tag on the principal for ABAC                                | String    |
| `aws:ResourceTag`            | Match a tag on the resource for ABAC                                 | String    |
| `aws:RequestTag`             | Match tags passed in the API request                                 | String    |
| `aws:TokenIssueTime`         | When temporary credentials were issued                               | Date      |
| `aws:UserId`                 | The unique ID of the principal (not the friendly name)               | String    |
| `aws:ViaAWSService`          | True if the request was made by an AWS service on your behalf        | Bool      |
| `cloudformation:SourceAccount` | The account ID that owns the CloudFormation stack or stack set — prevents confused deputy in cross-account operations | String |
| `cloudformation:SourceArn`     | The ARN of the CloudFormation stack or stack set — used with `cloudformation:SourceAccount` for confused deputy prevention | Arn    |

> ABAC — _Attribute-Based Access Control_ — granting access based on tags matching between principal and resource (e.g., `aws:PrincipalTag/Department` must equal `aws:ResourceTag/Department`).
>
> For more details see chapter [ABAC (Attribute-Based Access Control)](iam.md#16-abac-attribute-based-access-control).

### 4.2. Service-Specific Condition Keys

These apply only to specific AWS services:

| Condition Key                     | Service | Purpose                                            |
| --------------------------------- | ------- | -------------------------------------------------- |
| `s3:x-amz-server-side-encryption` | S3      | Require server-side encryption on uploads          |
| `s3:VersionId`                    | S3      | Restrict operations on specific object versions    |
| `s3:signatureversion`             | S3      | Require a specific signature version (SigV4)       |
| `kms:ViaService`                  | KMS     | Only allow KMS usage through specific AWS services |
| `kms:CallerAccount`               | KMS     | Restrict KMS key usage to a specific account       |
| `kms:EncryptionContextKeys`       | KMS     | Require specific encryption context keys           |
| `ec2:InstanceType`                | EC2     | Restrict which EC2 instance types can be launched  |

### 4.3. Condition Operators

Condition operators define how the condition key is evaluated:

| Category   | Operators                                                                                                                         |
| ---------- | --------------------------------------------------------------------------------------------------------------------------------- |
| String     | `StringEquals`, `StringNotEquals`, `StringEqualsIgnoreCase`, `StringLike`, `StringNotLike`                                        |
| Numeric    | `NumericEquals`, `NumericNotEquals`, `NumericLessThan`, `NumericLessThanEquals`, `NumericGreaterThan`, `NumericGreaterThanEquals` |
| Date       | `DateEquals`, `DateNotEquals`, `DateLessThan`, `DateLessThanEquals`, `DateGreaterThan`, `DateGreaterThanEquals`                   |
| Bool       | `Bool` (checks true/false)                                                                                                        |
| Binary     | `BinaryEquals`                                                                                                                    |
| IP Address | `IpAddress`, `NotIpAddress`                                                                                                       |
| ARN        | `ArnEquals`, `ArnNotEquals`, `ArnLike`, `ArnNotLike`                                                                              |
| Null       | `Null` (checks whether the condition key exists)                                                                                  |
| Set        | `ForAllValues` (all values in the request must match), `ForAnyValue` (at least one value must match)                              |

## 5. IAM Roles

### 5.1. Role Anatomy

- **Trust policy**: Defines who can assume the role (Principal + Conditions)
- **Permissions policy**: Defines what the role can do once it is assumed
- When a role is assumed, AWS STS returns temporary credentials that the principal uses to make API calls

### 5.2. PassRole (`iam:PassRole`)

This is a critical exam concept. The `iam:PassRole` permission controls whether a user can pass a role to an AWS service.

- You must have `iam:PassRole` to attach a role to an AWS resource (EC2 instance, Lambda function, CloudFormation stack, etc.)
- The resource being launched receives the role's permissions
- Can be scoped down using the `iam:PassedToService` condition key to restrict which service the role can be passed to
- Can also restrict which specific roles can be passed by specifying the role ARN in the Resource element

**Exam scenario**: A developer can launch EC2 instances but gets an error when trying to attach a specific IAM role. The likely cause is missing `iam:PassRole` permission for that role ARN.

### 5.3. Service-Linked Roles

- Predefined roles linked directly to specific AWS services
- Created automatically by the service when you enable certain features
- Contain all permissions the service needs to operate
- Trust policy is locked to the service — you cannot modify who can assume them
- Cannot be deleted until the service no longer needs them
- **Not affected by SCPs** (important exam distinction)

### 5.4. Service Roles

- Roles that an AWS service assumes to perform actions on your behalf
- You create these roles and attach a trust policy that allows the service to assume them
- Example: An EC2 instance role, a Lambda execution role, or a CloudFormation service role

### 5.5. Instance Profiles

- A container that holds an IAM role for use with EC2 instances
- An EC2 instance can only have one instance profile attached at a time (which can contain one role)
- The AWS SDK and CLI automatically call `sts:AssumeRole` when using an instance profile
- Use instance profiles instead of storing access keys on EC2 instances

## 6. Permissions Boundaries

- A managed policy that sets the **maximum** permissions an IAM user or role can have
- Does NOT grant permissions by itself — it only **caps** them
- Effective permissions = (identity-based Allow minus explicit Deny) limited to the boundary Allow
- Supported for: IAM Users and IAM Roles only (NOT groups)
- Use case: Delegating admin access to let users create their own roles without giving them full administrative control

**Exam scenario**: An administrator creates a permissions boundary allowing only EC2 and S3 actions. A developer creates a role with an attached policy that grants IAM admin access. The role can still only perform EC2 and S3 actions because the boundary caps the effective permissions.

### 6.1. Permissions Boundaries vs SCPs

| Aspect            | Permissions Boundary                   | SCP                                   |
| ----------------- | -------------------------------------- | ------------------------------------- |
| Scope             | Single user or role                    | Entire account or OU                  |
| Who configures it | Account administrator                  | Organization management account       |
| Effect            | Caps individual identities             | Caps the entire account               |
| How to override   | Remove the boundary from the user/role | Remove the SCP from the OU or account |

## 7. Session Policies

- Passed when assuming a role or requesting federated credentials
- Supported by STS APIs: `AssumeRole`, `AssumeRoleWithWebIdentity`, `GetFederationToken`, `GetSessionToken`
- Passed using the `--policy-arns` parameter or inline policy
- The session policy and the identity policy are evaluated together — the more restrictive of the two applies
- Session policies can only RESTRICT — they can never grant new permissions beyond what the identity policy allows
- Effective permissions = common overlap of the identity policy, session policy, boundary, and SCP

## 8. STS (AWS Security Token Service)

### 8.1. Key APIs

| API                         | Use Case                                                                                                               |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `AssumeRole`                | Get temporary credentials for a role in the same or a different AWS account                                            |
| `AssumeRoleWithSAML`        | Get temporary credentials for a user authenticated via a SAML 2.0 identity provider                                    |
| `AssumeRoleWithWebIdentity` | Get temporary credentials for a user authenticated by a web identity provider (Cognito, Google, Facebook, Amazon)      |
| `GetSessionToken`           | Get temporary credentials for an IAM user, typically used to enforce MFA                                               |
| `GetFederationToken`        | Get temporary credentials for a federated user (does not create an AssumedRoleUser — the caller IS the federated user) |

### 8.2. Important Details

- Temporary credentials consist of: `AccessKeyId`, `SecretAccessKey`, and `SessionToken`
- Default expiry is 1 hour, maximum is 12 hours (for `AssumeRole`)
- **Regional STS endpoints** are available to improve latency and reliability:
  - Global endpoint: `sts.amazonaws.com` (default)
  - Regional endpoints: `sts.<region>.amazonaws.com`
- STS is a global service by default but can be constrained to operate only in specific regions
- `GetCallerIdentity` is a useful debugging tool — it returns the ARN of the principal making the call

### 8.3. Using MFA with STS

```bash
aws sts get-session-token \
  --serial-number arn:aws:iam::123456789012:mfa/user \
  --token-code 123456
```

- Returns temporary credentials with MFA context
- Use with `aws:MultiFactorAuthPresent` condition key in IAM policies to enforce MFA

### 8.4. Revoking Temporary Credentials

- `AWSRevokeOlderSessions` is an AWS managed policy that denies all actions from principals whose credentials were issued before a specific timestamp
- Attach this policy to a role or user to invalidate older sessions
- Use case: Incident response — when credentials may be compromised, attach this policy to immediately revoke all sessions issued before the compromise time

## 9. Cross-Account Access

### 9.1. Role-Based Cross-Account Access

1. Account A creates an IAM role with a trust policy that allows principals from Account B to assume it
2. Account B grants its users permission to call `sts:AssumeRole` targeting the role ARN in Account A
3. A user in Account B assumes the role and receives temporary credentials scoped to the role's permissions

### 9.2. External ID

- Prevents the **confused deputy** problem — a situation where an entity without permissions can coerce a more privileged entity to perform actions on its behalf
- Required when a third party needs cross-account access to YOUR account
- The external ID is a unique value agreed upon by both parties
- Must match in the trust policy AND in the `AssumeRole` API call
- **Exam critical**: The external ID is NOT a secret — it is a unique identifier that prevents one customer from accidentally or maliciously assuming a role intended for another customer

**Trust policy example**:

```json
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::222222222222:root"
    },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {
        "sts:ExternalId": "unique-id-12345"
      }
    }
  }
}
```

### 9.3. Resource-Based Policies

Some AWS services natively support resource-based policies, which grant cross-account access directly without requiring role assumption:

- Services: S3 (bucket policies), SQS (queue policies), SNS (topic policies), KMS (key policies), Lambda (function policies), Secrets Manager (secret policies), Glue (catalog policies), ECR (repository policies)
- No need to assume a role — the resource policy grants access directly to the calling principal
- When using resource-based policies, the evaluated principal is the **caller's identity**, not the resource owner's identity

### 9.4. Choosing the Right Pattern

| Scenario                                    | Recommended Approach                               |
| ------------------------------------------- | -------------------------------------------------- |
| A single service needs cross-account access | Resource-based policy (if the service supports it) |
| A human or role needs cross-account access  | Cross-account IAM role                             |
| A third-party vendor needs access           | Cross-account role with External ID                |
| Many resources and services need access     | Cross-account role as a single access point        |

## 10. IAM Access Analyzer

### 10.1. Key Features

- Analyzes resource policies to identify resources that are shared with external principals outside your zone of trust
- **Zone of Trust** = your AWS account or your AWS Organization
- Any access granted outside the zone of trust generates a finding
- Supported resources: S3 buckets, IAM roles, KMS keys, Lambda functions, SQS queues, Secrets Manager secrets, SNS topics, EBS snapshots, EFS file systems, Glue catalogs, RDS DB snapshots, Outposts resources

### 10.2. Access Analyzer Policy Generation

- Generates least-privilege IAM policies based on actual CloudTrail access history
- Uses the last 90 days of CloudTrail logs to determine which actions and resources a role has accessed
- Located in the IAM console under Access Analyzer > Policy generation
- Outputs a refined JSON policy that you can review and apply

### 10.3. Exam Tips

- A finding that says "external access" means you should investigate whether the access is intentional or an over-permission
- Use in audit workflows to detect unintended cross-account resource exposure
- Integrates with AWS Security Hub and Amazon EventBridge for automated response workflows

## 11. IAM Policy Simulator

- Web-based tool that simulates IAM policy evaluation without making real API calls
- Supports all five policy types: identity-based, resource-based, SCPs, permission boundaries, and session policies
- Shows exactly which statement in which policy allowed or denied the simulated action
- Ideal for debugging why a principal is receiving AccessDenied errors

### 11.1. Troubleshooting AccessDenied

1. Check CloudTrail for the failed API call — review `errorCode` and `errorMessage`
2. Reproduce the scenario in the Policy Simulator
3. If explicit deny → check SCPs first, then identity-based Deny statements
4. If implicit deny → check if the action or resource is missing from Allow statements
5. Check Condition keys — a failed condition results in implicit deny
6. Check permission boundaries — the action might be granted by the identity policy but outside the boundary
7. Check session policies if temporary credentials are being used

## 12. IAM Access Advisor

- Shows when each AWS service was **last accessed** by a user, role, or group
- **Service Last Accessed**: Shows the last time a service was accessed (up to 400 days of history)
- **Action Last Accessed**: Shows the last time each individual action within a service was accessed (more granular)
- Use case: Identify permissions that have never been used, then create a more restrictive policy to follow least privilege
- Found in the IAM console under the "Access Advisor" tab for any user, role, or group

## 13. IAM Credential Report

- A CSV-formatted report that lists all IAM users in the account and the status of their credentials
- Includes: password status, password last used, password last changed, access key age, access key last used, MFA device status, and signing certificate status
- Can be downloaded from the IAM console or generated via the CLI: `aws iam generate-credential-report`
- Use case: Regularly audit for old access keys, users without MFA, and unused credentials

## 14. IAM Identity Center (formerly AWS SSO)

### 14.1. Purpose

- Centralized workforce identity and access management for multiple AWS accounts and business applications
- Formerly known as AWS Single Sign-On (SSO) — renamed but functionally the same service
- Supports multiple identity sources:
  - Built-in identity store
  - AWS Managed Microsoft AD
  - On-premises Active Directory (via AD Connector)
  - External SAML 2.0 identity provider

### 14.2. Permission Sets

- A collection of IAM policies assigned to a user or group for a specific AWS account
- Can include AWS managed policies, customer managed policies, and inline policies
- Assigned at the account level, not the user level
- **Exam tip**: Permission sets are implemented as IAM roles in the target account — AWS Identity Center creates and manages these roles automatically

### 14.3. Key Concepts

| Concept               | Description                                                                              |
| --------------------- | ---------------------------------------------------------------------------------------- |
| Identity Center Admin | Created in the AWS Organizations management account or a delegated administrator account |
| Permission Set        | A template of permissions that can be applied to one or more accounts                    |
| Assignment            | The binding of a user or group and a permission set to a specific AWS account            |
| SSO Login URL         | The portal URL where users sign in, formatted as `<your-id>.awsapps.com/start`           |

### 14.4. IAM Identity Center vs IAM Users

| Aspect                | IAM Identity Center                    | IAM Users                                                 |
| --------------------- | -------------------------------------- | --------------------------------------------------------- |
| Credential management | Centralized, no long-lived access keys | Per-user access keys and passwords                        |
| MFA enforcement       | Centralized policy                     | Per-user configuration                                    |
| Multi-account access  | One login, access to multiple accounts | Separate user needed per account                          |
| Best for              | Workforce access to multiple accounts  | Programmatic access, service accounts, machine identities |

## 15. Federation

### 15.1. SAML 2.0 Federation

- Establishes trust between AWS and an external SAML 2.0 identity provider (AD FS, Okta, Ping, etc.)
- User authenticates with the IdP and receives a SAML assertion
- The SAML assertion is exchanged for temporary AWS credentials via `AssumeRoleWithSAML`
- Steps:
  1. Configure an IAM SAML identity provider in the IAM console
  2. Create one or more IAM roles with a trust policy that allows the SAML provider to assume them
  3. User authenticates at the IdP portal
  4. IdP generates a SAML assertion and posts it to the AWS sign-in endpoint
  5. STS validates the assertion and returns temporary credentials mapped to the IAM role

### 15.2. OIDC / Web Identity Federation

- Uses `AssumeRoleWithWebIdentity` to exchange tokens from an OIDC identity provider for AWS credentials
- Supported identity providers: Amazon Cognito, Login with Amazon, Google, Facebook
- **Amazon Cognito** is the recommended AWS service for web identity federation:
  - Cognito Identity Pools: Exchanges identity tokens for IAM role credentials
  - Cognito User Pools: Handles user registration, sign-in, and authentication

### 15.3. Custom Federation

- Uses the `GetFederationToken` API to issue temporary credentials
- The caller must have `sts:GetFederationToken` permissions
- Returns temporary credentials with the caller's identity as the federated user
- No role assumption involved — the caller IS the federated user, not an assumed role

### 15.4. IAM Roles Anywhere

- Provides temporary AWS credentials for workloads running **outside** AWS
- Uses X.509 certificates to authenticate the workload
- Creates a trust relationship between the external workload and an IAM role
- Certificates can be issued by AWS Certificate Manager (ACM) Private CA or by your own certificate authority
- Use case: On-premises servers, non-AWS cloud workloads, or any workload that cannot use an instance profile

### 15.5. AWS Private Certificate Authority (PCA)

- Fully managed private CA service for creating and managing X.509 certificates
- Defines a private CA hierarchy (root CA → subordinate CAs) for issuing internal certificates
- Key integrations:
  - **IAM Roles Anywhere**: Serves as a trust anchor — certificates from PCA authenticate on-premises workloads to obtain temporary AWS credentials
  - **ACM**: Issue private certificates for internal-facing resources (ALB, API Gateway, CloudFront, EC2)
  - **API Gateway mTLS**: Validate client certificates from your private CA for mutual TLS
  - **AWS IoT**: Manage device certificates at scale
  - **Code Signing for Lambda**: Sign code with PCA certificates
- Security controls:
  - Root CA can be kept offline; subordinate CAs handle daily issuance
  - Certificate revocation via CRLs or OCSP
  - All CA operations logged to CloudTrail
  - IAM policies control who can issue, revoke, and manage certificates
- **Exam scenario**: Need private certificates for internal apps without managing CA infrastructure → **ACM Private CA**
- **Exam scenario**: IAM Roles Anywhere needs a trust anchor → **ACM Private CA** as trust anchor or your own external CA

## 16. ABAC (Attribute-Based Access Control)

### 16.1. How It Works

- Access decisions are based on **tags** attached to both the principal and the resource
- A principal's tag is compared to the resource's tag — if they match (based on the condition), access is granted
- Condition keys: `aws:PrincipalTag/<key>` and `aws:ResourceTag/<key>`

### 16.2. ABAC Example

```json
"Condition": {
  "StringEquals": {
    "aws:ResourceTag/Project": "${aws:PrincipalTag/Project}"
  }
}
```

This condition allows access only if the principal's `Project` tag matches the resource's `Project` tag.

### 16.3. ABAC vs RBAC

| Aspect              | RBAC (Role-Based)                               | ABAC (Attribute-Based)                 |
| ------------------- | ----------------------------------------------- | -------------------------------------- |
| Management overhead | Many roles needed for different combinations    | Fewer roles, access controlled by tags |
| Scaling             | Requires a new role for each new combination    | Just add a tag                         |
| Complexity          | Roles can become numerous and hard to manage    | Requires strong tag governance         |
| Best for            | Simple, static environments with few variations | Dynamic, fast-growing environments     |

### 16.4. ABAC Prerequisites (Exam Critical)

1. Enable tags on all resources — use SCPs to enforce tag-on-create
2. Define a tagging strategy with consistent key-value pairs
3. Ensure all principals carry the required tags
4. Create ABAC policies that match principal tags to resource tags
5. **Tag governance is the most important prerequisite** — without it, ABAC fails silently. Untagged resources are inaccessible (no tag match), and mistagged resources are accessible by the wrong team.

## 17. SCPs (Service Control Policies)

### 17.1. Overview

- Specify the **maximum** permissions for accounts in an AWS Organization
- Applied at the Root OU, any OU, or directly to an individual account
- Do NOT grant permissions — they only set a boundary
- Effective permissions = common overlap of SCP and IAM permissions
- Default: `FullAWSAccess` is attached to all roots and OUs. If you remove it without adding an Allow SCP, all actions are denied for all principals in that account.
- Deny always wins — even over an Allow SCP within the same policy
- Affect ALL principals in the account, including the root user
- Do NOT affect service-linked roles (important exam distinction)

### 17.2. Common SCP Patterns

```json
// Deny leaving the organization
{
  "Effect": "Deny",
  "Action": "organizations:LeaveOrganization",
  "Resource": "*"
}

// Enforce MFA for all actions
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "BoolIfExists": {
      "aws:MultiFactorAuthPresent": "false"
    }
  }
}

// Restrict allowed regions
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:RequestedRegion": ["us-east-1", "eu-west-1"]
    }
  }
}
```

### 17.3. Policy Layer Summary

| Layer               | Grants Permissions? | Caps Permissions? | Scope                           | Who Manages                     |
| ------------------- | ------------------- | ----------------- | ------------------------------- | ------------------------------- |
| SCP                 | No                  | Yes               | Entire account or OU            | Organization management account |
| Permission Boundary | No                  | Yes               | Single IAM user or role         | Account administrator           |
| Identity Policy     | Yes                 | No                | Single IAM user, group, or role | Account administrator           |
| Resource Policy     | Yes                 | No                | Single AWS resource             | Account administrator           |

## 18. KMS Key Policies and Grants

AWS KMS uses resource-based policies (key policies) to control access to KMS keys. Understanding how key policies interact with IAM policies and grants is critical for the exam.

### 18.1. Key Policy Basics

- Every KMS key has a **key policy** — a resource-based policy attached directly to the key that defines who can use it and how
- If the key policy allows IAM policies to grant access, then IAM policies in the account can also control access to the key
- Without a key policy allowing it, IAM policies alone **cannot** grant access to a KMS key
- **Exam scenario**: A user with full IAM `AdministratorAccess` cannot use a KMS key → check the key policy (it likely does not grant IAM policy-based access)

### 18.2. Key Policy vs IAM Policy

| Configuration | Behavior |
|---|---|
| Default key policy | Grants full control to account root user; allows IAM policies to manage access |
| Custom key policy with IAM access | Explicitly permits IAM policies to grant access in addition to the key policy |
| Custom key policy without IAM access | Only the principals listed in the key policy can use the key — IAM policies are ignored |

### 18.3. Grants

- Grants are an alternative to key policies for granting KMS key access
- More restrictive — allow specific operations for specific principals without modifying the key policy
- Created via the `CreateGrant` API
- **Use case**: Cross-account access without updating the key policy (the grantee's IAM policy must still allow the KMS action)
- Grants can be retired by the grantee at any time using `RetireGrant`
- Key difference: Grants are **operation-specific** and can include constraints (encryption context)

### 18.4. Key Condition Keys

| Condition Key | Purpose |
|---|---|
| `kms:ViaService` | Restrict KMS key usage to requests that originate through a specific AWS service (e.g., `s3.amazonaws.com`, `ec2.amazonaws.com`). **Heavily tested** — scope a key to a single service |
| `kms:CallerAccount` | Restrict key usage to a specific AWS account |
| `kms:EncryptionContextKeys` | Require specific encryption context keys in the KMS request |
| `kms:EncryptionContext` | Match the exact encryption context in the request |
| `kms:ReEncryptOnSameKey` | Restrict re-encryption to the same KMS key |
| `kms:GrantConstraintType` | Control which constraint types can be included in a grant |
| `kms:GrantOperations` | Limit which KMS operations a grant can allow |
| `kms:KeyOrigin` | Restrict based on key material origin (`AWS_KMS`, `EXTERNAL`, or `AWS_CLOUDHSM`) |
| `kms:ResourceAliases` | Restrict based on aliases attached to the key |

**kms:ViaService example** — restrict a key so it can only be used through S3:

```json
{
  "Effect": "Deny",
  "Action": "kms:*",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "kms:ViaService": "s3.amazonaws.com"
    }
  }
}
```

### 18.5. Cross-Account KMS Access

Two approaches:

1. **Key policy**: Add the external account or principal directly to the key policy
2. **Grant**: Use `CreateGrant` to allow a principal in another account to use the key (the external principal must also have the corresponding IAM permissions)

Both approaches require that the external account's IAM policies also allow the KMS actions.

## 19. RDS IAM Database Authentication

RDS IAM Database Authentication lets you use IAM roles and short-lived authentication tokens to connect to RDS instances instead of database passwords.

### 19.1. How It Works

- IAM generates a 15-minute authentication token signed with your AWS credentials
- The token is used as the database password — no passwords stored in application code or Secrets Manager
- Database access is managed centrally through IAM policies

### 19.2. Supported Databases

| Engine | Support |
|---|---|
| Amazon RDS MySQL | Yes |
| Amazon RDS PostgreSQL | Yes |
| Amazon RDS MariaDB | Yes |
| Amazon Aurora MySQL | Yes |
| Amazon Aurora PostgreSQL | Yes |
| Amazon RDS SQL Server, Oracle, Db2 | No |

### 19.3. Configuration Requirements

1. Enable **IAM DB Authentication** on the RDS instance
2. Create a database user with `AWSAuthenticationPlugin` (MySQL/MariaDB) or `rds_iam` (PostgreSQL)
3. Attach an IAM policy granting `rds-db:connect` to the principal that needs access
4. Use the AWS SDK or CLI to generate an auth token:

```bash
aws rds generate-db-auth-token \
  --hostname mydb.123456789012.us-east-1.rds.amazonaws.com \
  --port 3306 \
  --region us-east-1 \
  --username db_user
```

5. Connect using the token as the password with SSL/TLS enforced

### 19.4. IAM Policy Example

```json
{
  "Effect": "Allow",
  "Action": "rds-db:connect",
  "Resource": "arn:aws:rds-db:us-east-1:123456789012:dbuser:db-ABCDEFGHIJKL01234/db_user"
}
```

- Resource ARN format: `arn:aws:rds-db:<region>:<account>:dbuser:<resource-id>/<database-user-name>`
- The **resource ID** is the RDS resource ID (not the DB instance identifier) — find it via `aws rds describe-db-instances`

### 19.5. Exam Scenarios

- **No passwords in code**: Use RDS IAM auth with an EC2 instance profile or Lambda execution role
- **Centralized DB access control**: Manage database access through IAM instead of per-user DB credentials
- **15-minute token expiry**: Tokens auto-refresh via the AWS SDK
- **SSL is mandatory**: IAM DB authentication requires TLS connections
- **Cross-account access**: Supported — IAM principals in Account B can authenticate to an RDS instance in Account A

## 20. IAM Limits

| Resource                                      | Limit                             |
| --------------------------------------------- | --------------------------------- |
| IAM users per AWS account                     | 5,000                             |
| IAM roles per AWS account                     | 1,000                             |
| IAM groups per AWS account                    | 300                               |
| Managed policies per AWS account              | 1,500                             |
| Policies attached to a role                   | 20 total (10 managed + 10 inline) |
| Policy document size (managed)                | 6,144 bytes                       |
| Policy document size (inline for roles/users) | 10,240 bytes                      |
| Roles an instance profile can contain         | 1                                 |

## 21. Troubleshooting Scenarios

### 21.1. User Gets AccessDenied Despite Having an Allow Policy

- **Check SCPs** — an SCP at the organization level may include a Deny that overrides your Allow
- **Check permission boundaries** — the action may be allowed by the identity policy but outside the boundary's scope
- **Check Condition keys** — conditions like `aws:SourceIp`, `aws:MultiFactorAuthPresent`, or `aws:RequestedRegion` may not be satisfied
- **Check Resource ARN** — the policy might grant access to a different resource than the one being accessed
- **Check for explicit Deny statements** — there may be a Deny in another identity or resource policy

### 21.2. Cross-Account AssumeRole Fails

- Verify the trust policy in the target account allows the source principal
- Verify the source user or role has `sts:AssumeRole` permission on the target role ARN
- Check for External ID mismatch between the trust policy and the AssumeRole call
- Check SCPs in either account that might deny `sts:AssumeRole`
- Verify the role session name is valid

### 21.3. EC2 Instance Cannot Access S3

- Verify the EC2 instance has an IAM role attached (via an instance profile)
- Verify the role's policy grants the required S3 actions
- Check if there is an S3 bucket policy that denies the access
- If using an S3 VPC endpoint, verify the endpoint policy allows the access and route tables are configured correctly

### 21.4. Lambda Function Cannot Write to CloudWatch Logs

- Verify the Lambda execution role has `logs:CreateLogGroup`, `logs:CreateLogStream`, and `logs:PutLogEvents`
- Check the resource ARN — CloudWatch Logs requires specific ARN patterns
- Verify no permission boundary is blocking the CloudWatch Logs actions

## 22. IAM Best Practices for the Exam

### 22.1. Least Privilege

- Start with the minimum permissions needed and add more only as required
- Use IAM Access Advisor to identify unused permissions and tighten policies
- Use IAM Access Analyzer to generate least-privilege policies from actual usage
- Prefer specific resource ARNs over `"Resource": "*"`
- Prefer specific action names over `"Action": "s3:*"`

### 22.2. MFA

- Require MFA for all human users with AWS access
- Use the `aws:MultiFactorAuthPresent` condition key in IAM policies to enforce MFA
- Enable MFA on the root user and store the device in a secure location

### 22.3. Access Keys

- Rotate access keys regularly (every 90 days is the recommended interval)
- Prefer IAM roles over long-lived access keys whenever possible (EC2 instance profiles, Lambda execution roles)
- Deactivate and delete unused access keys
- Use the IAM credential report to audit access key usage

### 22.4. Root User

- Only use the root user for account-level tasks that require it (changing the support plan, closing the account, registering as a seller in the Reserved Instance Marketplace)
- Enable MFA on the root user
- Do NOT create access keys for the root user
- Do NOT use the root user for any daily administration tasks

### 22.5. IAM Groups

- Manage permissions through groups rather than attaching policies directly to users
- Assign users to groups and attach policies to the groups
- This simplifies permission management and auditing

### 22.6. Password Policy

- Enforce a strong password policy (minimum length, complexity requirements)
- Require periodic password rotation
- Prevent password reuse

### 22.7. Monitoring

- Use AWS CloudTrail to record all IAM API calls
- Set up Amazon CloudWatch alarms for significant IAM events (policy changes, role assumption failures)
- Use IAM Access Analyzer for continuous detection of unintended access
- Use AWS Config rules to enforce IAM compliance

### 22.8. Cross-Account Best Practices

- Prefer cross-account roles over creating IAM users in multiple accounts
- Always use an External ID when giving third-party access to your account
- Use `aws:SourceOrgID` or `aws:SourceAccount` conditions in trust policies for additional verification

### 22.9. Credential Compromise Incident Response

1. Rotate or delete the compromised credentials immediately
2. Attach the `AWSRevokeOlderSessions` policy to invalidate all existing sessions
3. Review CloudTrail logs to determine what actions were taken by the compromised credentials
4. Delete or deactivate access keys if they were the source of the compromise

## 23. Exam Tips Summary

- SCP allows do NOT grant permissions — an identity-based policy is still required
- SCP deny cannot be overridden by any account administrator
- Permission boundaries and session policies both restrict only — they can never grant permissions
- ABAC's prerequisite is strong tag governance (enforce tag-on-create with SCPs)
- The External ID is not a secret — it is a unique identifier that prevents the confused deputy problem
- `iam:PassRole` is required when attaching a role to a service and can be scoped using the `iam:PassedToService` condition
- Prefer temporary credentials from STS over long-lived access keys
- **Access Advisor** = shows last used services | **Access Analyzer** = finds external access | **Policy Simulator** = tests policies without making real API calls
- `aws:SourceOrgID` is preferred over listing individual account IDs in trust policies
- Service-linked roles are NOT affected by SCPs
- `AWSRevokeOlderSessions` invalidates sessions issued before a specific timestamp — useful during incident response
- MFA can be enforced using the `aws:MultiFactorAuthPresent` Bool condition together with `GetSessionToken`
