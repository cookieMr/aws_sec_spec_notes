# AWS Systems Manager (SSM)

<figure>
  <img src="./images/Arch_AWS-Systems-Manager_64@5x.png" alt="AWS Systems Manager Icon" width=200>
  <figcaption><center>AWS Systems Manager<br><i>Image source: AWS Documentation</i></center></figcaption>
</figure>

**Overview**: AWS Systems Manager (SSM) is a management service that provides secure remote access, patch management, configuration management, automation, and inventory tracking for EC2 instances and on-premises servers. For the Security Specialty exam, Session Manager, Patch Manager, Automation, and Parameter Store are the most relevant capabilities.

**Domain weight**: SSM appears across the Threat Detection and Incident Response domain (~14%), Infrastructure Security domain (~20%), and Management and Security Governance domain (~14%). It is the primary tool for secure instance access and automated remediation.

## 1. SSM Agent

- The SSM Agent is software installed on managed instances (EC2 or on-premises)
- Required for: Session Manager, Patch Manager, Run Command, Automation, State Manager, Inventory
- **Pre-installed** on most recent Amazon Linux, Ubuntu, and Windows Server AMIs
- For older or custom AMIs, you must install it manually
- The agent must have **outbound internet access** or a **VPC endpoint** to reach SSM endpoints
- The agent checks in with the SSM service to get commands and report status

**Exam scenario**: An EC2 instance is not appearing in the SSM console → check that the **SSM Agent** is installed, running, and has network access to the SSM endpoint (via internet or VPC endpoint).

## 2. Session Manager

### 2.1. Purpose

- Provides secure, audited shell access to EC2 instances and on-premises servers
- **No SSH needed** — no bastion host, no SSH keys, no security group rules for SSH (port 22)
- Uses IAM policies for access control
- All sessions are logged to CloudTrail and optionally to CloudWatch Logs or S3

**Exam scenario**: A security team wants to eliminate SSH bastion hosts and manage EC2 access through IAM → use **SSM Session Manager** for secure, keyless, audited shell access.

### 2.2. Key Security Features

| Feature              | Benefit                                                                      |
| -------------------- | ---------------------------------------------------------------------------- |
| **No inbound ports** | Instances do not need SSH (port 22) or RDP (port 3389) open                  |
| **No bastion host**  | Eliminates an entire attack surface                                          |
| **IAM-based access** | Control exactly which users can access which instances                       |
| **Full audit trail** | Every session is logged to CloudTrail (who accessed which instance, when)    |
| **Session logs**     | Session activity can be recorded to CloudWatch Logs or S3                    |
| **Port forwarding**  | Secure tunnel for accessing applications on instances without exposing ports |

### 2.3. How It Works

1. User initiates a session via AWS Console, CLI, or SDK
2. SSM authenticates the user via IAM
3. SSM sends a message to the SSM Agent on the target instance
4. SSM Agent establishes a bidirectional connection over HTTPS
5. User gets a shell on the instance — no network-level access to the instance needed

### 2.4. IAM Permissions

- The **user** needs: `ssm:StartSession`
- The **instance** (via instance profile) needs: `ssmmessages:CreateControlChannel`, `ssmmessages:CreateDataChannel`, `ssmmessages:OpenControlChannel`, `ssmmessages:OpenDataChannel`
- The instance profile also needs: `ssm:UpdateInstanceInformation`

## 3. Patch Manager

### 3.1. Purpose

- Automates patching of EC2 instances and on-premises servers
- Supports: Amazon Linux, Ubuntu, RHEL, SUSE, CentOS, Windows Server (including SQL Server)
- Uses **patch baselines** to define which patches are approved (by classification, severity, product)
- Can generate patch compliance reports

### 3.2. Key Concepts

| Concept                  | Description                                                           |
| ------------------------ | --------------------------------------------------------------------- |
| **Patch baseline**       | Defines which patches are approved for an OS type                     |
| **Patch group**          | A tag-based grouping of instances for staggered patching              |
| **Maintenance window**   | A schedule for when patching can occur                                |
| **Compliance reporting** | Reports which instances are compliant/non-compliant with the baseline |

### 3.3. Patch Baseline Rules

- Can define: auto-approve patches after X days, approve specific patch classifications (e.g., Critical, Security), reject specific patches
- **Default baselines**: AWS provides default baselines for each OS (approve all Critical and Security patches)
- **Custom baselines**: Create your own to enforce stricter policies

**Exam scenario**: A security policy requires all EC2 instances to have security patches applied within 7 days of release → configure **SSM Patch Manager** with a patch baseline that auto-approves security patches after 7 days, and schedule a **maintenance window** to apply them.

## 4. Automation

### 4.1. Purpose

- Run pre-defined or custom **runbooks** (SSM Automation documents) to automate common tasks
- Used extensively for **incident response** and **remediation** workflows
- Integrates with EventBridge, Config, and GuardDuty for automated response

### 4.2. Common Automation Documents

| Document                             | Use Case                              |
| ------------------------------------ | ------------------------------------- |
| `AWS-RunPatchBaseline`               | Apply patching                        |
| `AWS-StopEC2Instance`                | Stop an EC2 instance                  |
| `AWS-TerminateEC2Instance`           | Terminate an EC2 instance             |
| `AWS-CreateSnapshot`                 | Create an EBS snapshot (forensics)    |
| `AWS-RestartEC2Instance`             | Restart an EC2 instance               |
| `AWS-DisablePublicAccessForS3Bucket` | Remove public S3 access (remediation) |
| `AWS-UpdateCloudFormationStack`      | Update infrastructure                 |

### 4.3. Automation + Incident Response

GuardDuty finding → EventBridge → SSM Automation runbook → Isolate instance

Example flow:

1. GuardDuty detects `UnauthorizedAccess:EC2/C&CActivity.B`
2. EventBridge matches the finding
3. EventBridge triggers an SSM Automation runbook
4. The runbook:
   - Creates an EBS snapshot (forensics)
   - Detaches the instance from the Auto Scaling group
   - Changes the security group to isolate the instance
   - Sends an SNS notification

**Exam scenario**: A GuardDuty finding detects a compromised EC2 instance. The security team wants to automatically isolate it and take a forensic snapshot → use **SSM Automation runbook** triggered by EventBridge.

## 5. Run Command

### 5.1. Purpose

- Run ad-hoc commands or scripts on multiple instances at once
- No SSH needed — uses the SSM Agent
- Commands can be targeted by tags, resource groups, or instance IDs
- Results are returned and logged

### 5.2. Use Cases

- Run a shell script across a fleet of instances
- Install a security agent on all instances
- Check the status of a security configuration
- Run a compliance check across instances

## 6. State Manager

### 6.1. Purpose

- Keep EC2 instances in a defined state using **association** rules
- Ensures instances stay compliant with a desired configuration
- Can run on a schedule or when the instance is launched

### 6.2. Use Cases

- Ensure the SSM Agent is always running
- Ensure a security agent (e.g., CrowdStrike, Trend Micro) is installed and running
- Ensure a specific software package is installed
- Ensure a specific configuration file is in place

## 7. Parameter Store

### 7.1. Purpose

- Secure, hierarchical storage for configuration data and secrets
- Supports: strings, string lists, secure strings (encrypted with KMS)
- Integrates with EC2, Lambda, ECS, and CloudFormation
- No additional cost (standard parameters) — advanced and secure string parameters may have costs

### 7.2. Parameter Types

| Type             | Description          | Encryption     |
| ---------------- | -------------------- | -------------- |
| **String**       | Plain text value     | No             |
| **StringList**   | Comma-separated list | No             |
| **SecureString** | Encrypted value      | KMS (required) |

### 7.3. Parameter Tiers

| Tier         | Max Size | Cost | Policies                                               |
| ------------ | -------- | ---- | ------------------------------------------------------ |
| **Standard** | 4 KB     | Free | No parameter policies                                  |
| **Advanced** | 8 KB     | Paid | Supports parameter policies (expiration, notification) |

### 7.4. Security Use Cases

- Store database passwords, API keys, and other secrets
- Reference secrets in CloudFormation without hardcoding
- Rotate secrets using Lambda integration (custom, not automatic like Secrets Manager)
- Use KMS to encrypt SecureString parameters

**Exam scenario**: An application running on EC2 needs to access database credentials without hardcoding them in the source code → store the credentials in **SSM Parameter Store** as a SecureString and retrieve them at runtime.

### 7.5. Parameter Store vs Secrets Manager

| Feature                  | Parameter Store                  | Secrets Manager                                   |
| ------------------------ | -------------------------------- | ------------------------------------------------- |
| **Max secret size**      | 4 KB (Standard), 8 KB (Advanced) | 64 KB                                             |
| **Automatic rotation**   | No (requires Lambda)             | Yes (built-in)                                    |
| **Cross-account access** | Yes (with resource policies)     | Yes (with resource policies)                      |
| **Cost**                 | Free (Standard), Paid (Advanced) | Paid ($0.40/secret/month + rotation)              |
| **KMS encryption**       | Yes (SecureString)               | Yes                                               |
| **Use case**             | Config data, small secrets       | Database credentials, API keys requiring rotation |

**Exam tip**: If the scenario requires **automatic secret rotation**, use Secrets Manager. If the scenario just needs a simple parameter or secret without rotation, Parameter Store is sufficient.

## 8. Inventory

### 8.1. Purpose

- Collect software inventory, OS configuration, and instance metadata from managed instances
- Collects: installed applications, OS versions, network configuration, Windows patches, etc.
- Data is stored in S3 and can be queried via Athena

### 8.2. Security Use Cases

- Discover instances running outdated software
- Identify instances with missing security agents
- Audit installed applications for unauthorized software
- Compliance reporting for software inventory

## 9. Compliance

### 9.1. Purpose

- Reports patch compliance status for managed instances
- Shows which instances are compliant/non-compliant with patch baselines
- Integrates with Inspector (Inspector discovers CVEs, Patch Manager remediates them)

## 10. SSM VPC Endpoints

- SSM requires network connectivity between instances and the SSM service
- Options:
  - **Internet**: Instance has outbound internet (NAT Gateway or public IP)
  - **VPC Endpoints**: Interface endpoints for SSM services (recommended for private subnets)

Required endpoints for Session Manager:

- `com.amazonaws.<region>.ssm` (SSM)
- `com.amazonaws.<region>.ssmmessages` (Session Manager messaging)
- `com.amazonaws.<region>.ec2messages` (EC2 agent messaging)

**Exam scenario**: EC2 instances in a private subnet need to be managed by SSM but have no internet access → create **VPC Interface Endpoints** for SSM, SSMMessages, and EC2Messages.

## 11. IAM Roles for SSM

| Role                                     | Purpose                                                                       |
| ---------------------------------------- | ----------------------------------------------------------------------------- |
| **AmazonSSMManagedInstanceCore**         | Basic SSM management permissions: Run Command, Session Manager, Patch Manager |
| **AmazonSSMFullAccess**                  | Full SSM administrative access for users                                      |
| **AmazonSSMRoleForAutomationAssumeRole** | Trust role for SSM Automation documents to assume other roles                 |

The managed instance policy includes:

- `ssm:UpdateInstanceInformation`
- `ssmmessages:*` (for Session Manager)
- `ec2messages:*` (for agent messaging)

## 12. SSM vs Other Services

| Service                  | What It Does                                                   |
| ------------------------ | -------------------------------------------------------------- |
| **SSM Session Manager**  | Secure shell access without SSH                                |
| **SSM Patch Manager**    | Automated patching                                             |
| **SSM Automation**       | Automated runbooks for remediation                             |
| **SSM Parameter Store**  | Secret and configuration storage                               |
| **Systems Manager**      | Umbrella service for all the above                             |
| **EC2 Instance Connect** | Temporary SSH key access (different approach — still uses SSH) |

## 13. Exam Tips

1. **Session Manager replaces SSH bastions** — no inbound ports, no SSH keys, no bastion hosts. All sessions are logged to CloudTrail.

2. **SSM Agent** is required on all managed instances. Pre-installed on most modern AMIs.

3. **VPC Endpoints** are required for private subnets — SSM, SSMMessages, EC2Messages interface endpoints.

4. **Patch Manager** automates patching using patch baselines and maintenance windows.

5. **Automation runbooks** are used for incident response remediation (GuardDuty → EventBridge → Automation).

6. **Parameter Store** stores configuration data and secrets. Use SecureString with KMS encryption for secrets.

7. **Parameter Store vs Secrets Manager**: Parameter Store for simple config data (free for Standard tier). Secrets Manager for secrets that need automatic rotation.

8. **Run Command** runs ad-hoc scripts across multiple instances without SSH.

9. **State Manager** keeps instances in a desired state (ensures security agents are running, etc.).

10. **Inventory** collects software and configuration metadata for compliance reporting.

11. **SSM Automation + Config**: Config rules can trigger SSM Automation for automatic remediation of non-compliant resources.

12. **SSM Automation + GuardDuty**: GuardDuty findings can trigger SSM Automation runbooks for automatic incident response.

13. **No inbound ports needed** for Session Manager — the connection is initiated by the SSM Agent (outbound to AWS).

14. **IAM policies control access** — granular control over who can start sessions on which instances.

15. **Session logging** should be enabled to CloudWatch Logs or S3 for audit trail and compliance.
