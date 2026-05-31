# AWS Organizations & SCPs

<figure>
  <img src="./images/aws-organizations.png" alt="AWS Organization Icon" width=200>
  <figcaption><center>AWS Organization<br><i>Image source: <a href="https://vecta.io/symbols/23/aws-security-identity-compliance/7/aws-organizations">Vecta.io</a></i></center></figcaption>
</figure>

**Overview**: AWS Organizations is the governance foundation for multi-account environments. It enables you to centrally manage billing, control access through Service Control Policies (SCPs), and automate account creation. The exam treats Organizations as the control plane for all multi-account security strategies — you must understand SCP inheritance, evaluation logic, and common guardrail patterns.

**Domain weight**: Tested within the IAM domain (~20%) and Infrastructure Security domain (~18%) of SCS-C03.

## 1. AWS Organizations Fundamentals

### 1.1. Account Types

| Account Type                         | Description                                          | Permissions                                                                                                   |
| ------------------------------------ | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| Management Account (formerly Master) | The account that created the organization            | Full administrative control over the entire organization. Cannot be changed without deleting the organization |
| Member Account                       | Any account added or created within the organization | Subject to SCPs applied to its OU or directly to the account                                                  |

### 1.2. Organization Hierarchy

```
Management Account
  └── Root (parent container for all accounts)
        ├── OU: Security
        │     ├── Account: Audit
        │     └── Account: Log Archive
        ├── OU: Workloads
        │     ├── OU: Production
        │     │     ├── Account: Prod-App
        │     │     └── Account: Prod-DB
        │     └── OU: Development
        │           ├── Account: Dev-App
        │           └── Account: Dev-Test
        └── Account: Sandbox (directly under Root, no OU)
```

- **Root** is the top-level container (NOT the AWS account root user — confusingly named)
- **Organizational Units (OUs)** are logical groupings of accounts
- Policies (SCPs, tag policies, backup policies) are attached to Root, OUs, or individual accounts
- Accounts inherit policies from every OU above them in the hierarchy

### 1.3. Consolidated Billing

- All member accounts receive the consolidated billing benefit without any additional configuration
- Volume discounts (S3, EC2, Lambda, etc.) are aggregated across all accounts
- Reserved Instance and Savings Plan sharing can be enabled across accounts
- Billing is managed by the management account, but each account retains its own billing data

### 1.4. OrganizationAccess Role

- When you invite an existing account to join your organization, the account administrator can create the `OrganizationAccess` IAM role
- This role grants the management account full access to the member account
- The role's trust policy allows the management account's IAM principals to assume it
- Without this role, the management account cannot access the member account directly (must rely on IAM roles in the member account)

## 2. Service Control Policies (SCPs)

### 2.1. What SCPs Do

- SCPs specify the **maximum** permissions for all principals in an account
- They do NOT grant any permissions — they only set a boundary
- Effective permissions = SCP ∩ IAM (the intersection of SCP allows and IAM allows)
- SCPs affect ALL principals in the account, including the AWS account root user
- SCPs do NOT affect service-linked roles, AWS-managed internal service principals, or the management account

### 2.2. SCP Effects on Different Principals

| Principal Type                      | Affected by SCP?                                |
| ----------------------------------- | ----------------------------------------------- |
| IAM users                           | Yes                                             |
| IAM roles                           | Yes (including assumed role sessions)           |
| AWS account root user               | Yes                                             |
| IAM Identity Center permission sets | Yes (they create roles in member accounts)      |
| Service-linked roles                | **No** — SCPs never apply                       |
| AWS service principals (internal)   | No                                              |
| Management account                  | No — SCPs never apply to the management account |

### 2.3. Default SCP Behavior

- When you enable SCPs in an organization, a default `FullAWSAccess` policy is attached to the Root and all OUs
- `FullAWSAccess` allows every action — it sets no restrictions
- If you detach `FullAWSAccess` without attaching an allow-list SCP, **all actions are denied** (implicit deny)
- There are two approaches to SCPs:
  - **Deny list**: Attach `FullAWSAccess` and add Deny SCPs for the services/actions to block
  - **Allow list**: Detach `FullAWSAccess` and attach SCPs that only Allow the approved services

### 2.4. SCP Inheritance

- SCPs applied at the Root level cascade to all OUs and accounts
- SCPs applied at an OU are inherited by all child OUs and their accounts
- SCPs applied directly to an account affect only that account
- If multiple SCPs apply (from Root + OU + account), the effective SCP is the intersection of ALL allows

**Inheritance example**:

```
Root SCP: Allow EC2, S3, IAM
  └── OU: Production SCP: Add Allow Lambda
        └── Account: Prod-App
              Effective: EC2, S3, IAM, Lambda (all allowed)

Root SCP: Allow EC2, S3, IAM
  └── OU: Restricted SCP: Deny IAM
        └── Account: Restricted-App
              Effective: EC2, S3 (IAM denied by the OU-level SCP)
```

### 2.5. SCP Evaluation Logic

1. All applicable SCPs are collected (Root + OU chain + account-level)
2. If ANY SCP has an explicit `Deny` for the action → **explicit deny** (cannot be overridden)
3. If an Allow-list approach is used and no SCP grants an Allow → **implicit deny**
4. If there is an Allow in at least one SCP and no Deny → **allowed** (then IAM policies are evaluated separately)

**Key point**: SCPs and IAM policies are evaluated independently at different layers. An SCP Allow does NOT mean the action is granted — the IAM policy must also Allow it.

## 3. Common SCP Patterns for the Exam

### 3.1. Deny Leaving the Organization

```json
{
  "Effect": "Deny",
  "Action": "organizations:LeaveOrganization",
  "Resource": "*"
}
```

- Prevents member accounts from leaving the organization
- Should be applied at the Root level so it affects all accounts
- The management account is not affected by SCPs, so this does not prevent it from managing the org

### 3.2. Enforce MFA

```json
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
```

- Denies all actions if the principal did not authenticate with MFA
- `BoolIfExists` ensures that long-lived credentials (which lack the MFA context key entirely) are also denied
- This is one of the most commonly tested SCP patterns

### 3.3. Restrict AWS Regions

```json
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

- Limits which AWS regions can be used by all accounts under the OU
- **Important**: Some global services (IAM, CloudFront, Route 53, STS global endpoint) operate outside of regions and will be denied if the condition key does not apply
- Use `aws:RequestedRegion` — some services (like CloudFront, IAM) do not have a region, so the condition key may not be present
- Use `Null` condition to check if the region key exists before applying the restriction

### 3.4. Block Disabling Security Services

```json
{
  "Effect": "Deny",
  "Action": [
    "guardduty:DeleteDetector",
    "guardduty:UpdateDetector",
    "cloudtrail:StopLogging",
    "cloudtrail:DeleteTrail",
    "config:DeleteConfigRule",
    "config:StopConfigurationRecorder"
  ],
  "Resource": "*"
}
```

- Prevents member accounts from disabling security monitoring
- Common pattern for SOC 2, PCI-DSS, and other compliance frameworks
- Paired with organizational trails and delegated GuardDuty admin

### 3.5. Restrict EC2 Instance Types

```json
{
  "Effect": "Deny",
  "Action": "ec2:RunInstances",
  "Resource": "arn:aws:ec2:*:*:instance/*",
  "Condition": {
    "StringNotEquals": {
      "ec2:InstanceType": ["t3.small", "t3.medium"]
    }
  }
}
```

- Restricts which EC2 instance types can be launched across the organization
- Helps control cost and enforce compliance

### 3.6. Require Encryption on S3

```json
{
  "Effect": "Deny",
  "Action": "s3:PutObject",
  "Resource": "*",
  "Condition": {
    "Null": {
      "s3:x-amz-server-side-encryption": "true"
    }
  }
}
```

- Requires that all S3 uploads include server-side encryption
- Prevents accidental storage of unencrypted objects

### 3.7. Prevent IAM Users and Access Keys

```json
{
  "Effect": "Deny",
  "Action": ["iam:CreateUser", "iam:CreateAccessKey"],
  "Resource": "*"
}
```

- Enforces a policy of using IAM Identity Center and roles instead of IAM users with access keys
- Common in organizations adopting modern access management patterns

### 3.8. Restrict Actions to Approved Services

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ec2:*", "s3:*", "lambda:*", "logs:*", "cloudwatch:*"],
      "Resource": "*"
    }
  ]
}
```

- Allow-list approach: only the listed services are permitted
- Requires detaching `FullAWSAccess` from the OU/account
- All other services are implicitly denied

## 4. SCPs vs Other Policy Layers

| Layer               | Grants? | Caps? | Affected Principals                                               | Scope               | Manager                         |
| ------------------- | ------- | ----- | ----------------------------------------------------------------- | ------------------- | ------------------------------- |
| SCP                 | No      | Yes   | All principals except service-linked roles and management account | Account or OU       | Organization management account |
| Permission Boundary | No      | Yes   | Specific IAM user or role                                         | Individual identity | Account administrator           |
| Identity Policy     | Yes     | No    | Specific IAM user, group, or role                                 | Individual identity | Account administrator           |
| Resource Policy     | Yes     | No    | Any principal specified                                           | Individual resource | Account administrator           |

## 5. AWS Organizations Features (Non-SCP)

### 5.1. Tag Policies

- Enforce consistent tagging across the organization
- Define which tags are required and their allowed values
- Accounts that violate the tag policy are flagged as non-compliant
- Usable with AWS Config to detect and remediate violations

### 5.2. Backup Policies

- Centrally define backup plans for resources across the organization
- Automate backup frequency, retention, and copy rules
- Enforce backup compliance across all accounts

### 5.3. AI Services Opt-Out Policies

- Control whether AWS AI services can use your organization's content for service improvement
- Applied at the organization level to all accounts

### 5.4. Delegated Administration

- The management account can delegate administration of specific AWS services (GuardDuty, Security Hub, CloudTrail, etc.) to a member account
- The delegated admin account can manage the service across all accounts in the organization
- **Exam scenario**: A security team wants a member account to manage GuardDuty without giving them the management account. Use delegated administration.

### 5.5. Organization CloudTrail (Organization Trail)

- A CloudTrail trail that logs events for all accounts in the organization
- Created in the management account, automatically applies to all member accounts
- Member accounts can see the trail but cannot modify or delete it
- Logs are delivered to a single S3 bucket (usually in the Log Archive account)
- Ensures immutable audit log collection — cannot be stopped by individual account admins

## 6. AWS Organizations and Security Services

### 6.1. GuardDuty

- Can be enabled across all accounts from a delegated administrator account
- Findings from all member accounts are visible in the delegated admin account
- SCPs should block disabling GuardDuty

### 6.2. Security Hub

- Centralized security findings aggregation across accounts
- Enabled from the delegated administrator account
- Cross-account aggregation allows viewing all findings in one place
- SCPs should block disabling Security Hub controls

### 6.3. AWS Config

- Multi-account aggregation using an aggregator in a central account
- AWS Config rules can be enforced across accounts
- Organization Config rules are automatically applied to new accounts

### 6.4. IAM Identity Center

- Must be enabled in the management account
- Provides single sign-on across all accounts in the organization
- Permission sets create IAM roles in each account
- Managed from a delegated admin account or the management account

## 7. SCP Limits

| Resource                                                 | Limit                             |
| -------------------------------------------------------- | --------------------------------- |
| Maximum SCPs per organization                            | 1,000                             |
| Maximum SCPs attached to one node (Root, OU, or account) | 5                                 |
| Maximum size of one SCP                                  | 5,120 characters                  |
| Maximum policy document size (including all statements)  | 5,120 characters                  |
| Maximum number of OUs                                    | 1,000                             |
| Maximum number of accounts                               | 5,000 (default), can be increased |

## 8. SCP Testing and Troubleshooting

### 8.1. SCP Behavior Testing

- SCP changes take effect **immediately** — there is no propagation delay
- Test SCPs in a dedicated testing OU before applying to production OUs
- Use the IAM Policy Simulator to test SCP effects — it supports SCP evaluation
- Regularly review effective permissions using IAM Access Analyzer

### 8.2. Common SCP Issues

**Issue: A member account administrator cannot perform an action even with full IAM permissions**

- Check SCPs applied to their OU — an SCP Deny or missing Allow is likely the cause
- Verify the account is not in the management account (SCPs do not apply to the management account)
- Check if the action is performed by a service-linked role (SCPs do not apply)

**Issue: Some global services are blocked by a region restriction SCP**

- Services like IAM, CloudFront, Route 53, and the STS global endpoint operate in `us-east-1` globally or have no region context
- The `aws:RequestedRegion` condition key may not be present for these services
- Add an exception in the SCP for these services using `NotAction` or a separate Allow statement

**Issue: SCP Allow does not grant expected access**

- Remember: SCPs only cap, they do not grant. The IAM identity policy must also Allow the action
- If the identity policy does not Allow the action, the SCP Allow makes no difference

### 8.3. SCP Inheritance Troubleshooting

- The effective SCP for an account is the union of all Deny statements and the intersection of all Allow statements from all ancestors
- A Deny at ANY level in the hierarchy causes the action to be denied (explicit deny)
- An Allow at the Root level can be overridden by a Deny at a child OU level
- To verify effective SCPs, use the AWS Organizations console's "Effective SCPs" view for an account

## 9. Moving Accounts Between OUs

- Accounts can be moved between OUs at any time
- When an account is moved, SCPs from the new OU apply immediately
- There is no downtime, but the account's effective permissions change instantly based on the new OU's SCPs
- The management account is the only account that can move accounts between OUs

## 10. Exam Tips

- SCPs do NOT affect the management account — ever. This is a critical distinction
- Service-linked roles are NOT affected by SCPs — they continue to work even under restrictive SCPs
- SCPs can restrict the AWS account root user — this is one of the few ways to control the root user
- SCPs do NOT grant permissions — they only set a maximum boundary
- The default `FullAWSAccess` SCP allows everything — remove it to implement allow-list SCPs
- SCP changes take effect immediately, not after a propagation delay
- Organization trails (CloudTrail) are the preferred pattern for immutable audit logging across accounts
- Delegated administration allows the management account to offload service management without granting access to the management account itself
- `aws:PrincipalOrgID` and `aws:SourceOrgID` are condition keys that work across accounts in the organization — use them instead of listing individual account IDs
- SCPs are evaluated before IAM policies, but the effective permission is the intersection of both
- Use the **deny-list** approach for most scenarios (attach Deny SCPs to block specific actions)
- Use the **allow-list** approach only when you need to tightly restrict an OU to a few approved services (e.g., a sandbox OU)
- Tag policies and SCPs can work together — use tag policies to enforce tagging and SCPs to enforce security guardrails
- SCPs cannot restrict the management account from performing administrative actions in member accounts (if the `OrganizationAccess` role exists)
