# AWS IAM Identity Center

<figure>
  <img src="./images/Arch_AWS-IAM-Identity-Center_64@5x.png" alt="AWS IAM Identity Center Icon" width=200>
  <figcaption><center>AWS IAM Identity Center <br><i>Image source: AWS Documentation</i></center></figcaption>
</figure>

**Overview**: AWS IAM Identity Center (formerly AWS Single Sign-On / AWS SSO) is the recommended AWS service for managing workforce identities and granting access to multiple AWS accounts and applications. It replaces the legacy practice of creating IAM users in every account. The exam treats Identity Center as the modern, preferred pattern for workforce identity management, and expects you to understand how it differs from IAM users, how permission sets work, and how it integrates with AWS Organizations.

**Domain weight**: Tested within the IAM domain (~20% of SCS-C03).

## 1. What Is IAM Identity Center?

- A centralized identity and access management service for workforce users
- Renamed from AWS Single Sign-On (SSO) — the service and API names still sometimes reference "SSO"
- Best for: granting human users access to multiple AWS accounts and business applications
- NOT for: machine-to-machine access, service accounts, or CI/CD roles (use IAM roles for those)
- Must be enabled in the AWS Organizations management account
- Supports up to 20,000 users in the built-in identity store

## 2. Identity Sources

Identity Center can authenticate users from three types of identity sources:

| Identity Source              | Description                                                 | Best For                                            |
| ---------------------------- | ----------------------------------------------------------- | --------------------------------------------------- |
| Built-in identity store      | Identity Center's own directory, managed within the service | Small teams, standalone AWS environments            |
| AWS Managed Microsoft AD     | Managed Active Directory in AWS                             | Existing AD infrastructure, Windows workloads       |
| On-premises Active Directory | Connect via AD Connector or two-way forest trust            | Hybrid environments with existing on-prem AD        |
| External SAML 2.0 IdP        | Okta, Azure AD, Ping, OneLogin, etc.                        | Enterprises with an existing SAML identity provider |

### 2.1. Identity Provider Synchronization

- For external IdPs, Identity Center does NOT store user passwords — authentication happens at the IdP
- User and group information can be synchronized from the IdP into Identity Center via SCIM (System for Cross-domain Identity Management) protocol
- SCIM sync enables automatic provisioning of users and groups
- When a user is removed from the IdP, SCIM removes their access in Identity Center automatically
- **Exam tip**: If a user's access needs to be centralized, use an external IdP with SCIM sync for automatic provisioning and de-provisioning

### 2.2. Active Directory Integration

- Two options:
  - **AWS Managed Microsoft AD**: Identity Center directly integrates with the directory
  - **On-premises AD Connector**: Proxy requests to on-prem AD without caching credentials in AWS
- Users and groups from AD are automatically available in Identity Center
- Changes in AD (password resets, account disabling) take effect immediately

## 3. Permission Sets

### 3.1. What Permission Sets Are

- A permission set is a template of permissions that can be assigned to users and groups for specific AWS accounts
- Each permission set is implemented as an **IAM role** in each target account
- Permission sets can include:
  - AWS managed policies (e.g., `AdministratorAccess`, `ReadOnlyAccess`)
  - Customer managed policies (created in each target account)
  - Inline policies (defined directly in the permission set)
  - Permission boundaries (optional, to further restrict the effective permissions)

### 3.2. How Permission Sets Work

1. An administrator creates a permission set in the Identity Center console
2. The permission set is assigned to a user or group for one or more AWS accounts
3. Identity Center creates an IAM role in each target account with the policies defined in the permission set
4. The role's trust policy allows the Identity Center service to assume it
5. When the user signs in, Identity Center authenticates them and calls `sts:AssumeRole` on their behalf
6. The user receives temporary credentials scoped to the permission set's policies

### 3.3. Permission Set vs IAM Role

| Aspect              | Permission Set                            | IAM Role                   |
| ------------------- | ----------------------------------------- | -------------------------- |
| Managed by          | IAM Identity Center administrator         | Account IAM administrator  |
| Created             | In Identity Center console                | In IAM console             |
| Underlying resource | Creates/maintains IAM roles automatically | The IAM role itself        |
| Scope               | Assigned to accounts OUs                  | Exists in a single account |
| Session duration    | Configurable (15min–12h)                  | Set on the role            |
| MFA                 | Centralized in Identity Center            | Per-role trust policy      |

### 3.4. Inline vs Managed Policies in Permission Sets

- Managed policies referenced in a permission set must exist in each target account with the same name and content
- If a customer managed policy is referenced, Identity Center creates it in each target account automatically
- Inline policies are embedded directly in the permission set — no dependency on pre-existing policies
- **Exam tip**: If you need the same policy across many accounts, use an inline policy in the permission set to avoid creating the policy manually in each account

## 4. Multi-Account Access

### 4.1. Assignment Model

- A permission set assignment is a three-way binding:
  - **Principal**: User or group in Identity Center
  - **Permission Set**: The template of permissions
  - **Target**: One or more AWS accounts (or OUs, which apply to all accounts in the OU)
- A single user can have different permission sets for different accounts

### 4.2. AWS Account Assignment

```text
User: alice@example.com
  ├── Account: Prod (Permission Set: ReadOnlyAccess)
  ├── Account: Dev (Permission Set: PowerUserAccess)
  └── Account: Security (Permission Set: SecurityAudit)
```

- Alice signs in once and can switch between accounts without re-authenticating
- Each account is accessed via a separate IAM role that Identity Center manages

### 4.3. Assigning to OUs vs Accounts

- Permission sets can be assigned directly to an AWS account or to an OU
- When assigned to an OU, all accounts in the OU (including future accounts) inherit the assignment
- **Exam tip**: Assigning permission sets to OUs is the preferred pattern — it automatically covers new accounts
- When assigned to an account, only that specific account gets the permission set

## 5. Authentication and User Portal

### 5.1. User Sign-In Flow

1. User navigates to the Identity Center portal URL: `https://<your-id>.awsapps.com/start`
2. User authenticates against the configured identity source (built-in, AD, or external IdP)
3. Identity Center determines which AWS accounts and applications the user has access to
4. User sees a dashboard with access buttons for each account/application
5. Clicking an AWS account triggers a role assumption via STS
6. The user is redirected to the AWS console with temporary credentials

### 5.2. Portal Customization

- Custom logo, banner, and support information can be configured
- The portal URL is `<directory-id>.awsapps.com/start`
- Users can access the portal from any device with a browser

### 5.3. Command Line Interface (CLI) Access

- Users can get temporary credentials for CLI access using:
  ```bash
  aws sso login --profile MyProfile
  ```
- The credential file is populated with temporary credentials that expire
- The SSO token provider stores a refresh token that allows silent credential refresh
- Credential process: `aws sso get-role-credentials` is called internally

## 6. MFA in Identity Center

### 6.1. MFA Configuration

- MFA is configured at the Identity Center level, not per-account
- Three MFA modes:

| Mode                  | Description                                                                                                                                      |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| Disabled              | No MFA required                                                                                                                                  |
| Enabled (recommended) | Users must register an MFA device; MFA is required for all access                                                                                |
| Context-aware         | MFA is required based on context (e.g., IP address, device, or application). Users can optionally skip MFA when accessing from trusted locations |

- MFA methods supported:
  - Authenticator apps (TOTP)
  - Security keys (FIDO2/WebAuthn)
  - Built-in authenticators (Windows Hello, Apple Touch ID, etc.)

### 6.2. MFA and Access Control

- MFA context in Identity Center is NOT the same as `aws:MultiFactorAuthPresent` in IAM conditions
- Identity Center MFA happens **before** role assumption
- The IAM role assumed by Identity Center does NOT carry the `aws:MultiFactorAuthPresent` context key in all cases — this depends on the identity source

### 6.3. MFA with External IdPs

- When using an external SAML 2.0 IdP, MFA is managed by the IdP, not by Identity Center
- Identity Center trusts the IdP's authentication assertion
- The IdP should enforce MFA before issuing the SAML assertion

## 7. ABAC in Identity Center

### 7.1. Attribute-Based Access Control

- Identity Center supports ABAC by passing user attributes as session tags during role assumption
- Supported attributes come from:
  - Identity Center's built-in directory (email, username, path, etc.)
  - External IdP (custom SAML attributes)
  - AD (user attributes synchronized via SCIM)
- These attributes are passed as session tags when Identity Center assumes the role in the target account

### 7.2. ABAC Configuration Steps

1. In Identity Center, enable "Access Control Attributes" and define which attributes to use
2. Configure the attribute mapping (e.g., map "Department" from the identity store to an IAM session tag)
3. In the target account, create IAM policies that reference the session tags:

```json
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "aws:PrincipalTag/Department": "${aws:ResourceTag/Department}"
    }
  }
}
```

4. Assign the permission set with ABAC-enabled policies

### 7.3. ABAC vs Multiple Permission Sets

- Without ABAC: Create a separate permission set for each department (Admin, Finance, Engineering, etc.)
- With ABAC: Create one permission set with condition keys that match user attributes to resource tags
- ABAC scales better — new departments just need the right tags, not a new permission set

## 8. Application Access

### 8.1. Supported Applications

- Identity Center can grant access to:
  - AWS managed applications (Amazon QuickSight, Amazon SageMaker Studio)
  - SAML 2.0 enabled applications (Salesforce, Box, Slack, etc.)
  - OIDC enabled applications
  - Custom applications

### 8.2. Application Assignment

- Similar to account assignment: user/group + application = access granted
- When a user signs into the portal, they see both AWS accounts and applications
- For SAML applications, Identity Center generates a SAML assertion and redirects the user to the application

## 9. Delegated Administration

- The management account can delegate administration of Identity Center to a member account
- The delegated administrator account can manage users, groups, permission sets, and assignments
- Delegated administrators CANNOT:
  - Change the identity source
  - Delete the Identity Center instance
  - Change MFA settings
  - Enable or disable ABAC attributes
- **Exam tip**: Delegated administration allows the security team to manage day-to-day Identity Center operations from a member account without granting management account access

## 10. Temporary Credentials and Session Duration

### 10.1. Session Duration Configuration

- Permission sets have a configurable session duration: minimum 15 minutes, maximum 12 hours
- Default session duration: 1 hour
- The duration is set on the permission set, not per-account
- After the session expires, the user must re-authenticate through the portal

### 10.2. Temporary Credentials Flow

1. User authenticates at the Identity Center portal
2. Identity Center obtains an OAuth access token for the user session
3. When the user clicks an account, Identity Center calls `sts:AssumeRole` with its service role in the target account
4. The returned temporary credentials are stored in the browser session
5. The user accesses the AWS console using these credentials
6. When the session expires, Identity Center refreshes by re-assuming the role (if the user is still authenticated at the IdP)

## 11. Identity Center and CloudTrail

- All Identity Center administrative actions are logged in CloudTrail:
  - User creation, group membership changes
  - Permission set creation and modification
  - Account assignments
  - Sign-in events
- User sign-ins are logged under `sso.amazonaws.com` as the event source
- Role assumption events appear in the target account's CloudTrail as `AssumeRole` calls from Identity Center
- **Exam tip**: Investigate unauthorized access by checking both the Identity Center CloudTrail (who signed in) and the target account's CloudTrail (what actions were taken)

## 12. Identity Center and SCPs

- Permission sets create IAM roles in each target account
- Those roles ARE subject to SCPs applied to the account or its OU
- If an SCP denies an action, the user cannot perform it even if the permission set grants it
- **Exam tip**: Identity Center permissions and SCPs intersect — the effective permission is the common overlap of both

## 13. Identity Center vs IAM Users

| Aspect              | IAM Identity Center                         | IAM Users                                                 |
| ------------------- | ------------------------------------------- | --------------------------------------------------------- |
| Credential type     | Temporary (STS)                             | Long-lived access keys and passwords                      |
| Credential lifetime | Max 12 hours                                | Until manually rotated or deleted                         |
| MFA                 | Centralized configuration                   | Per-user configuration                                    |
| Account access      | One authentication, multiple accounts       | Separate user per account required                        |
| Best for            | Human workforce access to multiple accounts | Programmatic access, service accounts, machine identities |
| Password policy     | Managed centrally                           | Per-account password policy                               |
| Access keys         | None — uses STS                             | Long-lived access keys                                    |
| Account footprint   | One Identity Center instance for the org    | Users exist in each account separately                    |
| Auditing            | CloudTrail + portal access logs             | CloudTrail + credential report                            |

## 14. Identity Center Limits

| Resource                        | Limit                                  |
| ------------------------------- | -------------------------------------- |
| Users (built-in store)          | 20,000                                 |
| Groups (built-in store)         | 20,000                                 |
| Permission sets                 | 1,000                                  |
| Permission set policies per set | 10                                     |
| Accounts per assignment         | 10,000                                 |
| Groups per user                 | 2,000                                  |
| Concurrent active sessions      | Determined by AWS Organizations limits |

## 15. Exam Tips

- Identity Center is the **modern/preferred** pattern for workforce access — the exam expects you to choose it over IAM users for human access scenarios
- Permission sets are implemented as IAM roles in each target account — this is a critical architectural detail
- Assign permission sets to OUs, not individual accounts, to automatically cover new accounts
- Identity Center does NOT replace IAM roles — it creates and manages them automatically
- ABAC in Identity Center uses session tags passed during role assumption — requires the target account's IAM policies to use `aws:PrincipalTag` conditions
- Delegated administration allows member accounts to manage Identity Center without management account access
- MFA enforcement at the Identity Center level is simpler than configuring MFA in individual account trust policies
- Identity Center roles are subject to SCPs — the effective permissions are the intersection
- For CLI access, users run `aws sso login` which obtains temporary credentials
- CloudTrail records Identity Center sign-ins under `sso.amazonaws.com` and role assumptions in each target account
- When using an external IdP, SCIM sync is required for automatic user provisioning and de-provisioning
- Identity Center must be enabled in the management account — it cannot be enabled from a member account
- If you see a question about managing access for employees to multiple AWS accounts, Identity Center is the answer
