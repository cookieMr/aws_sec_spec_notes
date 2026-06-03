# AWS Directory Service

<figure>
  <img src="./images/Arch_AWS-Directory-Service_64@5x.png" alt="AWS Directory Service Icon" width=200>
  <figcaption><center>AWS Directory Service<br><i>Image source: AWS Documentation</i></center></figcaption>
</figure>

**Overview**: AWS Directory Service provides managed Microsoft Active Directory in the AWS Cloud. It enables directory-aware workloads, facilitates Single Sign-On (SSO) to AWS applications, and integrates on-premises Active Directory with AWS. The exam tests understanding of the three directory options, trust relationships, and which option fits a given scenario.

**Domain weight**: Directory Service appears within the IAM domain (~20% of SCS-C03) and in questions about hybrid identity, federation, and workload integration.

## 1. Three Directory Options

| Option                       | What It Is                                                                                         | Underlying Tech   | Supports Trusts     | Supports MFA           | Supports LDAPS     | RDS SQL Server Compatible |
| ---------------------------- | -------------------------------------------------------------------------------------------------- | ----------------- | ------------------- | ---------------------- | ------------------ | ------------------------- |
| **AWS Managed Microsoft AD** | Fully managed actual Microsoft AD in AWS. Two domain controllers across AZs.                       | Windows Server AD | Yes                 | Yes (RADIUS or native) | Yes                | Yes                       |
| **AD Connector**             | Proxy service that forwards auth requests to your on-premises AD. No directory data stored in AWS. | Proxy only        | N/A (uses existing) | Yes (RADIUS)           | Depends on on-prem | No                        |
| **Simple AD**                | Standalone Samba 4 based directory for basic AD needs. Low-cost, low-scale.                        | Samba 4           | No                  | No                     | No                 | No                        |

**Key exam point**: Choosing the right directory type for a given scenario is the most commonly tested concept.

## 2. AWS Managed Microsoft AD

### 2.1. Overview

- Fully managed, highly available pair of domain controllers deployed across two Availability Zones
- Powered by Windows Server 2019
- AWS handles host monitoring, recovery, data replication, snapshots, and software updates
- Up to 30,000 directory objects (Standard Edition) or 500,000 (Enterprise Edition)
- Supports: Group Policy, Kerberos SSO, LDAPS, schema extensions, trust relationships, MFA

### 2.2. Editions

| Edition                | Max Directory Objects | Use Case                                           |
| ---------------------- | --------------------- | -------------------------------------------------- |
| **Standard Edition**   | ~30,000               | Small and midsize businesses up to 5,000 employees |
| **Enterprise Edition** | ~500,000              | Large enterprises needing broad directory support  |

### 2.3. Trust Relationships

- AWS Managed Microsoft AD can establish **forest trusts** with on-premises AD or AD in another AWS account
- Trust types: one-way (incoming/outgoing) or two-way (bidirectional)
- Users from the on-premises domain can access AWS resources without synchronizing credentials
- The trust provides seamless SSO across environments
- Requires network connectivity between VPC and on-premises network (VPN or Direct Connect)

**Exam scenario**: A company has on-premises AD and wants EC2 instances in AWS to authenticate users against the on-premises directory → create an **AWS Managed Microsoft AD** and establish a **two-way forest trust** with the on-premises AD.

### 2.4. Multi-Region Replication

- You can deploy additional domain controllers in other AWS Regions
- Provides low-latency authentication for users in different geographic regions
- Supports disaster recovery scenarios
- All domain controllers replicate directory data between regions
- Each region pair adds cost

### 2.5. LDAPS (LDAP over SSL/TLS)

- AWS Managed Microsoft AD supports LDAPS for secure LDAP communications
- You can use either:
  - AWS-provided certificate (automatically renewed)
  - Your own certificate from a Certificate Authority (CA)
- LDAPS uses TCP port 636
- **Exam scenario**: An application requires encrypted LDAP connections to authenticate users → verify that **AWS Managed Microsoft AD with LDAPS enabled** is used (Simple AD does not support LDAPS).

### 2.6. Shared Directory

- Share your AWS Managed Microsoft AD with other AWS accounts within the same organization
- The shared directory becomes available in the target account for joining EC2 instances, etc.
- The directory owner retains full administrative control
- Useful for multi-account architectures where a central identity directory is shared

### 2.7. Schema Extensions

- You can extend the AD schema to support custom attributes
- Extensions are synced to all domain controllers in the directory
- Required for applications that need custom AD attributes (e.g., Microsoft Exchange, SharePoint)

### 2.8. MFA

- Two MFA options:
  - **RADIUS-based MFA**: Integrate with existing RADIUS infrastructure (e.g., Azure MFA, Duo)
  - **Native MFA**: AWS Managed Microsoft AD can enforce MFA natively for certain AWS applications
- MFA can be enforced when users access AWS applications from the internet

### 2.9. Password Policies

- Manage password policies (complexity, length, expiration, lockout) using standard AD tools
- Policies apply consistently across all domain controllers in the directory
- Can be configured via Group Policy

### 2.10. Snapshots and Restore

- AWS takes **daily snapshots** automatically
- Manual snapshots can also be taken on demand
- Snapshots can be restored to create a new directory (cannot restore in-place — snapshot restores to a new directory)
- Useful for disaster recovery and testing

## 3. AD Connector

### 3.1. Overview

- A proxy service — **no directory data is stored in AWS**
- Forwards authentication requests from AWS applications to your on-premises AD
- Two sizes: **Small** (up to 500 users) and **Large** (up to 5,000 users)
- Requires one service account in your on-premises AD
- Runs in your VPC across two subnets in different AZs

### 3.2. Key Characteristics

- **No directory data** in AWS — user credentials never leave your on-premises AD
- **No synchronization** needed — users and groups are read live from on-premises AD
- You continue managing AD on-premises as before (password policies, lockouts, etc.)
- Supports **seamless domain join** for EC2 Windows instances to your on-premises domain
- Supports **RADIUS-based MFA**

### 3.3. What AD Connector Works With

| Supported                                        | Not Supported                           |
| ------------------------------------------------ | --------------------------------------- |
| Amazon WorkSpaces                                | RDS SQL Server                          |
| Amazon WorkDocs                                  | Trust relationships                     |
| Amazon QuickSight                                | LDAPS (relies on on-prem)               |
| Amazon Chime                                     | Schema extensions                       |
| Amazon Connect                                   | PowerShell AD cmdlets (on the AWS side) |
| Amazon WorkMail                                  |                                         |
| AWS Management Console (via IAM Identity Center) |                                         |
| EC2 Windows (seamless domain join)               |                                         |

### 3.4. When to Use AD Connector

**Exam scenario**: You want your existing on-premises AD users to access Amazon WorkSpaces and the AWS Management Console without synchronizing directory data to the cloud → use **AD Connector**. It proxies authentication requests to your on-premises domain controllers.

**Exam scenario**: You need RDS SQL Server with Windows Authentication → AD Connector **does not work**. You must use **AWS Managed Microsoft AD**.

## 4. Simple AD

### 4.1. Overview

- Low-cost, basic Active Directory compatible service powered by **Samba 4**
- Standalone directory in the cloud (not connected to on-premises AD)
- Two sizes: **Small** (up to 500 users) and **Large** (up to 5,000 users)
- Supports basic AD features: user accounts, groups, group policies, Kerberos SSO, joining EC2 instances to the domain

### 4.2. Limitations (All Testable on Exam)

| Feature                                                | Simple AD Support |
| ------------------------------------------------------ | ----------------- |
| Trust relationships                                    | No                |
| MFA                                                    | No                |
| LDAPS                                                  | No                |
| DNS dynamic update                                     | No                |
| Schema extensions                                      | No                |
| PowerShell AD cmdlets                                  | No                |
| FSMO role transfer                                     | No                |
| RDS SQL Server                                         | No                |
| AWS Management Console sign-in via IAM Identity Center | No                |

- Simple AD is a standalone directory — it cannot be connected to on-premises AD
- Samba 4 compatibility may not support all Active Directory aware applications
- **Exam scenario**: If the question mentions trust relationships, MFA, LDAPS, or RDS SQL Server compatibility, **Simple AD is the wrong answer** — use AWS Managed Microsoft AD instead.

### 4.3. When to Use Simple AD

- Low-scale, low-cost workloads needing basic AD features
- Testing, development, or proof-of-concept environments
- Workloads that only need LDAP support for Linux applications
- Simple AD is compatible with: WorkSpaces, WorkDocs, QuickSight, WorkMail

## 5. Comparison Summary

| Feature                          | AWS Managed Microsoft AD | AD Connector       | Simple AD         |
| -------------------------------- | ------------------------ | ------------------ | ----------------- |
| Managed by AWS                   | Yes                      | Yes                | Yes               |
| Actual Microsoft AD              | Yes                      | No (proxy)         | No (Samba 4)      |
| Maximum objects                  | 30k / 500k               | N/A (uses on-prem) | 500 / 5,000 users |
| Trust relationships              | Yes                      | No                 | No                |
| MFA                              | Yes (RADIUS or native)   | Yes (RADIUS only)  | No                |
| LDAPS                            | Yes                      | Depends on on-prem | No                |
| Schema extensions                | Yes                      | No                 | No                |
| Multi-region replication         | Yes                      | No                 | No                |
| RDS SQL Server                   | Yes                      | No                 | No                |
| Shared directory (cross-account) | Yes                      | No                 | No                |
| EC2 domain join                  | Yes                      | Yes                | Yes               |
| WorkSpaces                       | Yes                      | Yes                | Yes               |

## 6. Integration with Other Services

### 6.1. Amazon EC2

- **Seamless domain join**: EC2 Windows instances can be automatically joined to a directory at launch
- Supports AWS Managed Microsoft AD, AD Connector, and Simple AD for domain join
- Linux instances can use LDAP/Kerberos authentication against AWS Managed Microsoft AD
- Security groups must allow appropriate ports (TCP/UDP 53 DNS, TCP/UDP 88 Kerberos, TCP 135 RPC, TCP 389 LDAP, TCP 445 SMB, TCP 636 LDAPS, TCP 49152-65535 RPC)

### 6.2. Amazon RDS for SQL Server

- RDS SQL Server can use **AWS Managed Microsoft AD** for Windows Authentication
- **AD Connector and Simple AD are NOT compatible** with RDS SQL Server
- Enables domain-authenticated access to SQL Server databases
- The database instance must be in a VPC with network connectivity to the directory

### 6.3. Amazon WorkSpaces

- All three directory types support WorkSpaces (but with different feature sets)
- AWS Managed Microsoft AD: full feature set including MFA, group policy, trusted domains
- AD Connector: proxy auth to on-prem AD, RADIUS MFA
- Simple AD: basic AD features for WorkSpaces

### 6.4. IAM Identity Center (AWS SSO)

- AWS Managed Microsoft AD can be the identity source for IAM Identity Center
- AD Connector can be used with IAM Identity Center to authenticate against on-prem AD
- Simple AD is NOT supported as an identity source for IAM Identity Center
- Enables SSO to multiple AWS accounts and business applications

### 6.5. AWS Management Console

- Users can sign in to the AWS Management Console using AD credentials (via IAM Identity Center or direct SAML)
- Supported with: AWS Managed Microsoft AD and AD Connector
- Not supported with Simple AD

## 7. Security Considerations

### 7.1. Network Security

- Directory must be deployed in a **VPC** across at least **two subnets in different AZs** for high availability
- Use **security groups** to restrict traffic to and from the domain controllers
- Do not expose domain controllers directly to the internet — deploy in private subnets
- For hybrid connectivity, use **VPN** or **AWS Direct Connect**
- Required ports between resources and directory: DNS, Kerberos, LDAP, LDAPS, SMB, RPC

### 7.2. Encryption

- **In transit**: LDAPS (LDAP over SSL/TLS) on port 636 for encrypted directory lookups
- **At rest**: AWS Managed Microsoft AD data is encrypted at rest
- Kerberos traffic is encrypted by default
- Use the AWS-provided certificate or upload your own for LDAPS

### 7.3. Monitoring and Auditing

- **CloudTrail**: Logs all Directory Service API calls (CreateDirectory, DeleteDirectory, EnableLDAPS, etc.)
- **CloudWatch**: Monitor directory metrics, set alarms for authentication failures
- **AWS Config**: Track directory configuration changes and compliance
- **Security Hub**: Aggregate Directory Service findings with other security services
- **VPC Flow Logs**: Monitor network traffic to and from domain controllers

### 7.4. Backup and Recovery

- Automatic daily snapshots for AWS Managed Microsoft AD and Simple AD
- Snapshots restore to a **new directory** — not in-place restoration
- AD Connector has no snapshots (no directory data in AWS)
- For disaster recovery, consider multi-region replication with AWS Managed Microsoft AD

### 7.5. Least Privilege

- Delegate directory administration to specific users/groups using standard AD delegation
- Directory Service API actions in AWS are controlled via IAM policies
- Separate duties: AWS IAM controls administrative API access; AD controls directory-level permissions

## 8. Limits and Quotas

| Resource                                          | Limit                |
| ------------------------------------------------- | -------------------- |
| AWS Managed Microsoft AD max objects (Standard)   | ~30,000              |
| AWS Managed Microsoft AD max objects (Enterprise) | ~500,000             |
| AD Connector max users (Small)                    | 500                  |
| AD Connector max users (Large)                    | 5,000                |
| Simple AD max users (Small)                       | 500                  |
| Simple AD max users (Large)                       | 5,000                |
| Directories per AWS account (soft limit)          | 20                   |
| Shared directories per AWS account (soft limit)   | 10 received, 10 sent |

## 9. Directory Service vs Other Identity Services

| Need                                                 | Service                  |
| ---------------------------------------------------- | ------------------------ |
| Application user identities (web/mobile)             | Amazon Cognito           |
| Workforce identity with SSO to multiple AWS accounts | IAM Identity Center      |
| Managed Microsoft AD in the cloud                    | AWS Managed Microsoft AD |
| Proxy to on-premises AD for AWS applications         | AD Connector             |
| Basic standalone AD in the cloud                     | Simple AD                |

## 10. Exam Tips

1. **AWS Managed Microsoft AD vs AD Connector vs Simple AD**: The most common Directory Service question. Choose AWS Managed Microsoft AD for full AD features, trusts, RDS SQL Server, LDAPS. Choose AD Connector to proxy to on-prem AD without storing data in AWS. Choose Simple AD for low-cost, basic standalone AD.

2. **Trust relationships**: If the question requires linking on-prem AD with AWS (without synchronization, live authentication), the answer involves **AWS Managed Microsoft AD with a forest trust** to on-prem AD. AD Connector also provides this but as a proxy — for actual trust relationships between two AD forests, use AWS Managed Microsoft AD.

3. **RDS SQL Server**: If the question mentions Windows Authentication for SQL Server → requires **AWS Managed Microsoft AD**. AD Connector and Simple AD are NOT compatible.

4. **No directory data in AWS**: If the requirement is that user credentials never leave on-premises → **AD Connector** (acts as a proxy only).

5. **LDAPS**: If encrypted LDAP is required → **AWS Managed Microsoft AD** with LDAPS enabled. Simple AD does not support LDAPS.

6. **MFA**: AWS Managed Microsoft AD supports RADIUS-based MFA and native MFA. AD Connector supports RADIUS MFA only. Simple AD does NOT support MFA.

7. **Simple AD limitations**: No trusts, no MFA, no LDAPS, no schema extensions, no RDS SQL Server, no IAM Identity Center integration. Memorize this list — Simple AD is often a distractor answer.

8. **Multi-region**: If low-latency authentication is needed across regions → deploy additional domain controllers with **AWS Managed Microsoft AD** (multi-region replication).

9. **Shared directory**: To share a directory across AWS accounts in the same organization → use the **shared directory** feature of AWS Managed Microsoft AD.

10. **Snapshots**: Daily automatic snapshots for AWS Managed Microsoft AD and Simple AD. Snapshots restore to a new directory (not in-place).

11. **EC2 domain join**: All three directory types support joining EC2 Windows instances to the domain. Seamless domain join is available at instance launch.

12. **Network connectivity**: Directory requires VPC, multi-AZ deployment, and proper security group rules for DNS, Kerberos, LDAP/LDAPS, and RPC ports.

13. **CloudTrail**: All Directory Service management events are logged. Audit directory creation, deletion, and configuration changes.

14. **IAM Identity Center**: If using AD as the identity source for IAM Identity Center, the supported options are AWS Managed Microsoft AD and AD Connector — **not** Simple AD.
