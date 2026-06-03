# AWS Firewall Manager

<figure>
  <img src="./images/Arch_AWS-Firewall-Manager_64@5x.png" alt="AWS Firewall Manager Icon" width=200>
  <figcaption><center>AWS Firewall Manager<br><i>Image source: AWS Documentation</i></center></figcaption>
</figure>

**Overview**: AWS Firewall Manager is a central management service that lets you configure and monitor firewall rules across accounts and resources in AWS Organizations. It provides a single pane of glass to deploy WAF rules, Shield Advanced protections, Network Firewall policies, security groups, and DNS Firewall rules across all accounts in the organization.

**Domain weight**: Firewall Manager appears in the Infrastructure Security domain (~20% of SCS-C03) and Management and Security Governance domain. It is the answer for any scenario involving centralized firewall policy management across multiple accounts.

## 1. What Firewall Manager Does

| Capability                    | Description                                                                      |
| ----------------------------- | -------------------------------------------------------------------------------- |
| **Central policy management** | Define firewall rules once, apply to all accounts/ resources in the organization |
| **Automatic enforcement**     | New accounts and resources inherit policies automatically                        |
| **Compliance monitoring**     | Detect resources that do not comply with the policy                              |
| **Remediation**               | Automatically flag or remediate non-compliant resources                          |
| **Multi-account**             | Manage rules across all accounts in AWS Organizations                            |

## 2. Prerequisites

- Must use **AWS Organizations** with all features enabled
- Designate a **Firewall Manager administrator account** (from the management account)
- The administrator account can be the management account or a delegated member account

## 3. Supported Policy Types

### 3.1. WAF Policies

- Deploy **AWS WAF** web ACLs across accounts:
  - CloudFront distributions
  - Application Load Balancers
  - API Gateway APIs
  - AppSync GraphQL APIs
  - Cognito User Pools
- Can deploy **AWS Managed Rule Groups** (Common Rule Set, SQLi, Bot Control, etc.)
- Can deploy **custom rules** (IP sets, rate-based, geo match, etc.)
- New resources matching the policy criteria automatically get the WAF ACL

**Exam scenario**: A security team needs to ensure that every ALB in the organization has the same WAF rules applied → create a **Firewall Manager WAF policy** that automatically deploys the Web ACL to all ALBs in all accounts.

### 3.2. Shield Advanced Policies

- Centrally enable **Shield Advanced** across accounts
- Automatically associate Shield Advanced with:
  - CloudFront distributions
  - Route 53 hosted zones
  - ALBs, NLBs
  - Global Accelerator accelerators
  - EC2 Elastic IPs
- Enables consistent DDoS protection across the organization

**Exam scenario**: A security team wants Shield Advanced enabled on all CloudFront distributions across 50 accounts → create a **Firewall Manager Shield Advanced policy** that automatically associates Shield Advanced with all CloudFront distributions in the organization.

### 3.3. Network Firewall Policies

- Centrally deploy **AWS Network Firewall** policies across VPCs in the organization
- Define stateful and stateless rule groups
- Automate the deployment of Network Firewall in new VPCs and accounts
- Enforce consistent traffic inspection rules

### 3.4. Security Group Policies

Two types:

| Policy Type                | Description                                                        | Use Case                                                    |
| -------------------------- | ------------------------------------------------------------------ | ----------------------------------------------------------- |
| **Common security groups** | Deploy the same security group to all accounts                     | Ensure all accounts have a standard "SSH from bastion" rule |
| **Audit security groups**  | Define rules that are allowed or prohibited, then audit compliance | Detect security groups with public SSH access               |

- **Common security group**: Firewall Manager creates the security group in each account and enforces it
- **Audit security group**: Firewall Manager monitors existing security groups and reports non-compliant ones (does not modify them)

**Exam scenario**: The security team needs to ensure no security group in the organization allows SSH from 0.0.0.0/0 → use a **Firewall Manager audit security group policy** to detect violations across all accounts. Or use a **common security group policy** to deploy a pre-approved restrictive security group.

### 3.5. Route 53 Resolver DNS Firewall Policies

- Centrally manage domain list filtering across accounts
- Deploy DNS Firewall rules (allow lists, block lists)
- Enforce consistent DNS resolution policies across the organization
- Block access to known-bad domains or restrict to approved domains only

## 4. How Policies Work

### 4.1. Policy Components

| Component                      | Description                                                         |
| ------------------------------ | ------------------------------------------------------------------- |
| **Policy type**                | What kind of firewall rule (WAF, Shield, Network Firewall, SG, DNS) |
| **Resource type**              | What resources the policy applies to (e.g., ALB, CloudFront, VPC)   |
| **Exclude accounts/resources** | Optionally exclude specific accounts or resources                   |
| **Remediation action**         | Auto-remediate or flag non-compliant resources                      |

### 4.2. Automatic Enforcement

- When a new account is added to the organization, Firewall Manager automatically applies policies
- When a new resource is created (e.g., new ALB), Firewall Manager automatically associates the WAF ACL
- No manual intervention needed for new resources or accounts

### 4.3. Compliance Monitoring

- Firewall Manager provides a dashboard showing compliance status across accounts
- Non-compliant resources are listed with details about the violation
- Automatic remediation can be enabled for some policy types

## 5. Firewall Manager Administrator Account

- Designated from the **management account** of AWS Organizations
- Has full control over firewall policies
- Can be the management account itself or a delegated member account
- All policy changes are logged to CloudTrail
- Members cannot override or delete Firewall Manager policies

**Exam scenario**: A security team needs a delegated administrator account (not the management account) to manage firewall policies → designate a **member account** as the Firewall Manager administrator.

## 6. Firewall Manager vs Individual Service Configuration

| Approach               | Pros                                                             | Cons                                                          |
| ---------------------- | ---------------------------------------------------------------- | ------------------------------------------------------------- |
| **Firewall Manager**   | Central management, automatic enforcement, compliance monitoring | Requires AWS Organizations, management overhead to set up     |
| **Per-account config** | Simple for single accounts                                       | No central visibility, no automated enforcement, inconsistent |

**Exam tip**: If the scenario involves **multiple accounts** and the need for **consistent firewall rules**, the answer is Firewall Manager. For a single account, use the service directly (WAF, Shield, etc.)

## 7. Integration with Other Services

| Service                            | Integration                                                                          |
| ---------------------------------- | ------------------------------------------------------------------------------------ |
| **AWS Organizations**              | Foundation — Firewall Manager operates within Organizations                          |
| **WAF**                            | Centrally deploy and manage Web ACLs                                                 |
| **Shield Advanced**                | Centrally enable and associate Shield Advanced                                       |
| **Network Firewall**               | Centrally deploy network firewall policies                                           |
| **Security Groups**                | Common and audit security group policies                                             |
| **Route 53 Resolver DNS Firewall** | Centrally manage domain filters                                                      |
| **CloudTrail**                     | Logs all Firewall Manager API calls                                                  |
| **Security Hub**                   | Firewall Manager compliance findings can be forwarded to Security Hub                |
| **Config**                         | Config rules can detect non-compliant resources, Firewall Manager can remediate them |

## 8. Security Best Practices

- **Designate a dedicated Firewall Manager admin account** — do not use the management account for day-to-day operations
- **Use audit security group policies** to discover existing violations before enforcing common security groups
- **Enable automatic remediation** for WAF and Shield policies to ensure consistent protection
- **Monitor compliance dashboard** regularly for non-compliant resources
- **Use CloudTrail** to audit Firewall Manager policy changes
- **Start with WAF policies** — they are the most commonly tested and most impactful

## 9. Limits and Quotas

| Resource                               | Limit |
| -------------------------------------- | ----- |
| WAF policies per account               | 50    |
| Shield Advanced policies per account   | 1     |
| Network Firewall policies per account  | 50    |
| Security group policies per account    | 100   |
| Safety rules per security group policy | 10    |

## 10. Exam Tips

1. **Firewall Manager = central management for firewall rules across accounts.** If the scenario involves multiple AWS accounts and consistent firewall enforcement, the answer is Firewall Manager.

2. **Works only with AWS Organizations** — all features must be enabled. This is a prerequisite.

3. **Five policy types**: WAF, Shield Advanced, Network Firewall, Security Groups, Route 53 Resolver DNS Firewall.

4. **New resources and accounts are automatically covered** — no manual configuration needed after the policy is created.

5. **Security group policies have two modes**: **Common** (deploy the same SG to all accounts) and **Audit** (detect non-compliant SGs).

6. **Firewall Manager does not replace WAF or Shield** — it centrally manages them. You still need to configure the underlying services.

7. **Delegated administrator**: Can be the management account or a member account. The delegated admin manages all firewall policies.

8. **Members cannot override** Firewall Manager policies — they are enforced centrally.

9. **CloudTrail**: All Firewall Manager API calls are logged — audit policy changes centrally.

10. **Security Hub**: Firewall Manager compliance results can be sent to Security Hub.

11. **Shield Advanced via Firewall Manager**: Centrally enable Shield Advanced and associate it with resources across accounts.

12. **WAF via Firewall Manager**: Deploy the same Web ACL to all ALBs, CloudFront distributions, or API Gateway APIs across the organization.

13. **Remediation**: Firewall Manager can automatically remediate non-compliant resources (e.g., associate a WAF ACL with an unprotected ALB).

14. **One admin account** manages policies for the entire organization. Policy changes affect all accounts immediately.

15. **Firewall Manager vs SCPs**: SCPs control what actions are allowed at the API level. Firewall Manager controls firewall rule configuration. They are complementary — use SCPs to prevent removal of WAF ACLs and Firewall Manager to deploy WAF ACLs.
