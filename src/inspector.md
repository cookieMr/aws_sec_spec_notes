# Amazon Inspector

<figure>
  <img src="./images/Arch_Amazon-Inspector_64@5x.png" alt="Amazon Inspector Icon" width=200>
  <figcaption><center>Amazon Inspector<br><i>Image source: AWS Documentation</i></center></figcaption>
</figure>

**Overview**: Amazon Inspector is a vulnerability management service that scans AWS workloads for software vulnerabilities and unintended network exposure. Inspector continuously scans EC2 instances, ECR container images, and Lambda functions, identifying CVEs, CIS benchmark deviations, and network reachability risks.

**Domain weight**: Inspector appears in the Infrastructure Security domain (~20% of SCS-C03) and the Threat Detection domain. It is complementary to GuardDuty — Inspector finds vulnerabilities, GuardDuty detects active threats.

## 1. Inspector v2 (Current)

Inspector v2 is the current generation. **Inspector Classic is deprecated** and no longer available for new customers.

| Feature | Inspector v2 |
| ----------------------------------------------- | ------------------------------------------------------------ |
| Scan targets | EC2 instances, ECR container images, Lambda functions |
| Scan method | Agentless (for EC2 using SSM), agent-based (optional for deeper scans) |
| Trigger | Continuous (automatic when a new resource is created) |
| Vulnerability sources | CVE databases, CIS benchmarks |
| Risk score | Contextual risk scoring based on exploitability and network exposure |
| Cost | Per scanned resource per hour (no upfront licensing) |

## 2. Scan Targets

### 2.1. EC2 Instances

- **Network reachability**: Checks if EC2 instances are accessible from the internet on specific ports (SSH, RDP, etc.)
- **Host assessments**: Scans the OS and applications for CVEs and CIS benchmark deviations
- Requires the **SSM Agent** to be installed on the EC2 instance
- Uses the SSM Agent for agentless data collection (no Inspector-specific agent needed)
- Instances must have a public IP or be reachable via a VPC endpoint to SSM

**Exam scenario**: EC2 instances are not appearing in Inspector scans → check that the **SSM Agent** is installed and the instance has network access to the SSM endpoint.

### 2.2. ECR Container Images

- Scans container images in Amazon ECR repositories
- **Scan on push**: Automatically scans images when they are pushed
- **Continuous scan**: Re-scans images when new CVEs are published
- Scans for OS-level vulnerabilities and programming language dependencies
- Findings are associated with the image digest (specific version)

### 2.3. Lambda Functions

- Scans Lambda function code and dependencies for vulnerabilities
- Scans Lambda function configuration for security misconfigurations
- Automatically enables when Inspector is activated
- No agent required — Inspector analyzes the function package

## 3. Finding Types

| Finding Type | Description |
| -------------------------------- | ------------------------------------------------------------ |
| **CVE (Common Vulnerabilities and Exposures)** | Known software vulnerabilities in OS packages and libraries |
| **CIS benchmark deviation** | Non-compliance with CIS operating system security benchmarks |
| **Network reachability** | EC2 instance ports accessible from the internet |
| **Security best practices** | Lambda function configuration issues (e.g., overly permissive IAM role) |

### 3.1. Severity Levels

| Severity | Description |
| ------------ | ---------------------------------------------------------- |
| **Critical** | Vulnerability that is easily exploitable and has severe impact |
| **High** | Vulnerability that is exploitable with moderate effort |
| **Medium** | Vulnerability that requires specific conditions to exploit |
| **Low** | Vulnerability with limited impact or difficult exploit conditions |

### 3.2. Risk Score

- Inspector v2 assigns a **risk score** to each finding (1-10)
- The risk score considers:
  - CVE base score (CVSS)
  - Network accessibility of the resource
  - Exploitability (is there a known exploit?)
  - Whether the resource is internet-facing
- Two instances with the same CVE can have different risk scores — an internet-facing instance scores higher than an internal one

**Exam scenario**: Two EC2 instances have the same CVE but different risk scores → Inspector adjusts the risk based on **network accessibility** and **internet exposure** of each instance.

## 4. Inspector Classic (Deprecated)

- Older version of Inspector — **no longer available** for new customers
- Required an **Inspector agent** on EC2 instances
- Only scanned EC2 instances (no ECR or Lambda)
- Had a separate assessment template and run model (not continuous)
- If the exam mentions an "Inspector agent" or "assessment template" or "assessment run", it is referring to **Inspector Classic** — know that Inspector v2 does not work this way

## 5. Integration with Other Services

| Service | Integration |
| --------------- | ------------------------------------------------------------ |
| **Security Hub** | All Inspector findings are automatically forwarded to Security Hub |
| **EventBridge** | Findings are published as events for automated response |
| **Systems Manager** | SSM Automation documents can patch vulnerable instances |
| **Lambda** | Custom remediation workflows triggered by findings |
| **AWS Organizations** | Delegated administrator can manage Inspector across all accounts |

### 5.1. Inspector + Security Hub

- All Inspector findings appear in Security Hub automatically (when Security Hub is enabled)
- Provides centralized vulnerability management alongside GuardDuty, Config, and Macie findings
- Security Hub's CSPM (Cloud Security Posture Management) complements Inspector's workload scanning

### 5.2. Inspector + EventBridge

- New findings trigger EventBridge events
- Common automation: Finding → EventBridge → Lambda → SSM Patch → SNS notification
- Enables automated vulnerability remediation workflows

## 6. Multi-Account Management

- Designate a **delegated administrator** in AWS Organizations
- The delegated admin can enable Inspector for all member accounts
- Member accounts cannot disable Inspector if enabled by the delegated admin
- Findings from all accounts are visible to the delegated admin

**Exam scenario**: A security team wants to centrally manage vulnerability scanning across 50 accounts → enable Inspector at the organization level with a **delegated administrator**.

## 7. Network Reachability Scans

- Identifies EC2 instances that are accessible from the internet
- Checks specific ports that are commonly targeted (SSH:22, RDP:3389, HTTP:80, HTTPS:443)
- Reports which security group rules are allowing the internet access
- Does NOT test actual network connectivity — it analyzes security group rules and NACLs

**Exam scenario**: A security team wants to find all EC2 instances with SSH exposed to the internet → use **Inspector network reachability scans**.

## 8. Remediation Recommendations

- Each finding includes remediation guidance specific to the vulnerability
- For CVEs: Recommended package update version
- For network reachability: Recommended security group rule changes
- For CIS benchmarks: Specific configuration changes needed
- Remediation can be automated via Systems Manager Patch Manager or SSM Automation

## 9. Cost

- Inspector v2: Charged per resource per hour
  - EC2 instance scanning: per instance-hour
  - ECR image scanning: per image scan
  - Lambda scanning: per function scan
- No upfront licensing costs
- Activating Inspector in a region enables scanning for all supported resources — only scanned resources incur charges

## 10. Security Best Practices

- **Enable Inspector in all regions** — vulnerabilities exist everywhere
- **Use organization-level delegation** for multi-account management
- **Integrate with Security Hub** for centralized finding management
- **Automate remediation** using EventBridge + Systems Manager
- **Enable ECR scan on push** for container image security
- **Patch regularly** based on Inspector findings
- **Prioritize by risk score**, not just severity — an internet-facing instance with a medium CVE may be higher risk than an internal instance with a high CVE

## 11. Limits and Quotas

| Resource | Limit |
| ------------------------------------ | --------------------------- |
| Inspector assessment targets | Unlimited |
| Concurrent EC2 instance scans | Account-level (soft limit) |
| Concurrent ECR image scans | Account-level (soft limit) |
| Finding retention | Indefinite (until suppressed or expired) |

## 12. Inspector vs Other Security Services

| Service | What It Does |
| --------------- | ------------------------------------------------------------ |
| **Inspector** | Scans for vulnerabilities (CVEs, CIS benchmarks, network exposure) |
| **GuardDuty** | Detects active threats and malicious activity |
| **Config** | Tracks resource configuration compliance |
| **Macie** | Discovers sensitive data |
| **Security Hub** | Aggregates findings from all above services |

**Exam tip**: Inspector finds **vulnerabilities** (weaknesses that could be exploited). GuardDuty finds **threats** (active exploitation attempts). Config checks **compliance** (whether resources meet policy). They are complementary.

## 13. Exam Tips

1. **Inspector v2 is continuous and agentless** (uses SSM Agent). Inspector Classic required a dedicated agent and assessment runs.

2. **Three scan targets**: EC2 instances, ECR container images, and Lambda functions. Know what each covers.

3. **Network reachability** checks security group rules for internet exposure — it does not test actual network paths.

4. **Risk score** incorporates CVSS score + network exposure + exploitability. Two identical CVEs can have different risk scores based on resource context.

5. **SSM Agent** is required for EC2 scanning. If EC2 instances are not showing up, check SSM Agent status and network connectivity.

6. **Security Hub** receives all Inspector findings automatically. Both services should be enabled together.

7. **EventBridge** enables automated vulnerability remediation workflows.

8. **Inspector for ECR** supports scan-on-push and continuous re-scanning when new CVEs are published.

9. **Multi-account**: Use a delegated administrator to centrally manage Inspector across AWS Organizations.

10. **Inspector Classic** is deprecated. If the exam mentions "assessment templates" or "Inspector agents", know this is the old model. Inspector v2 is the current model.

11. **Complementary to GuardDuty**: Inspector finds vulnerabilities (what could go wrong). GuardDuty detects threats (what is actively going wrong).

12. **CIS benchmarks**: Inspector checks EC2 instances against CIS operating system benchmarks — useful for compliance reporting.

13. **Lambda scanning**: Inspector analyzes both function code/dependencies AND function configuration (IAM role, environment variables, etc.).

14. **No per-scan pricing**: Inspector v2 charges per resource-hour for continuous monitoring.

15. **Prioritize by risk score**: On the exam, if a question asks how to prioritize vulnerability remediation, the answer should reference Inspector's risk score (which accounts for exploitability and network exposure, not just CVSS).
