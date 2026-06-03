# Amazon GuardDuty

<figure>
  <img src="./images/Arch_Amazon-GuardDuty_64@5x.png" alt="Amazon GuardDuty Icon" width=200>
  <figcaption><center>Amazon GuardDuty<br><i>Image source: AWS Documentation</i></center></figcaption>
</figure>

**Overview**: Amazon GuardDuty is a continuous, intelligent threat detection service that monitors AWS accounts and workloads for malicious activity. It uses machine learning, anomaly detection, and integrated threat intelligence to identify suspicious behavior. GuardDuty is the primary threat detection service on the exam and the starting point for many incident response scenarios.

**Domain weight**: GuardDuty is the most important service in the Threat Detection and Incident Response domain (~14% of SCS-C03) and appears throughout Security Logging and Monitoring questions.

## 1. How GuardDuty Works

GuardDuty analyzes multiple data sources continuously:

| Data Source                      | What It Analyzes                                                  |
| -------------------------------- | ----------------------------------------------------------------- |
| **CloudTrail management events** | API calls that create, modify, or delete AWS resources            |
| **CloudTrail S3 data events**    | S3 object-level operations (GetObject, PutObject, ListObjects)    |
| **VPC Flow Logs**                | Network traffic metadata (IPs, ports, protocols)                  |
| **DNS logs**                     | DNS query patterns — detects C2 communication, data exfiltration  |
| **EKS audit logs**               | Kubernetes API calls within EKS clusters                          |
| **RDS login events**             | Database login attempts (RDS Protection)                          |
| **EBS volumes (malware scan)**   | Malware detection on EBS volumes (Malware Protection)             |
| **Runtime monitoring**           | OS-level activity on EC2, ECS, EKS (GuardDuty Runtime Monitoring) |

**Key exam point**: GuardDuty does NOT need agents or software — it consumes existing AWS logs. The one exception is **Runtime Monitoring** which deploys a security agent on EC2/ECS/EKS workloads.

## 2. Finding Types and Severity

### 2.1. Severity Levels

| Severity   | Score Range | Example                                                                        |
| ---------- | ----------- | ------------------------------------------------------------------------------ |
| **High**   | 7.0 - 8.9   | `UnauthorizedAccess:IAMUser/MaliciousIPCaller` — API calls from known-bad IP   |
| **Medium** | 4.0 - 6.9   | `Recon:IAMUser/UserPermissions` — enumerating IAM roles and policies           |
| **Low**    | 1.0 - 3.9   | `Discovery:S3/UnusualObjectListing` — listing S3 objects from unusual location |

- Findings are categorized by: **Threat purpose** (e.g., Backdoor, CryptoCurrency, Recon, UnauthorizedAccess, Stealth) and **Resource type** (IAMUser, Instance, EKS, S3, RDS)
- The full finding type follows the pattern: `<ThreatPurpose>:<ResourceType>/<ThreatName>`

### 2.2. Common Finding Types (Know These)

| Finding Type                                   | Severity | What It Detects                                         |
| ---------------------------------------------- | -------- | ------------------------------------------------------- |
| `Backdoor:EC2/C&CActivity.B`                   | High     | EC2 instance communicating with a C2 server             |
| `CryptoCurrency:EC2/BitcoinTool.B`             | Medium   | EC2 instance communicating with a cryptocurrency pool   |
| `Recon:IAMUser/UserPermissions`                | Medium   | IAM user enumerating their own permissions              |
| `Recon:EC2/PortProbeUnprotectedPort`           | Low      | External IP probing open ports on EC2                   |
| `UnauthorizedAccess:IAMUser/MaliciousIPCaller` | High     | API calls from an IP associated with malicious activity |
| `UnauthorizedAccess:EC2/SSHBruteForce`         | Medium   | SSH brute force attempts on EC2                         |
| `Behavior:IAMUser/ConsoleLoginSuccess.B`       | Low      | Console login from an unusual location                  |
| `Discovery:S3/UnusualObjectListing`            | Low      | Unusual S3 bucket listing activity                      |
| `Policy:IAMUser/RootCredentialUsage`           | High     | Root user credentials are being used                    |

## 3. Multi-Account Management

### 3.1. Delegated Administrator

- Designate a **GuardDuty delegated administrator** from the management account of AWS Organizations
- The delegated admin can:
  - Enable GuardDuty for all member accounts automatically
  - View findings from all accounts in a single console
  - Manage member accounts and suppression rules centrally
- Member accounts cannot disable GuardDuty if it was enabled by the delegated admin
- **Exam scenario**: A security team needs to centrally manage threat detection across 100 AWS accounts → designate a **GuardDuty delegated administrator** to enable and manage GuardDuty organization-wide.

### 3.2. Findings Aggregation

- The delegated administrator account sees all findings from all member accounts
- Member accounts see their own findings (and can see the delegated admin's findings if they choose)
- Findings are automatically forwarded to the delegated admin — no subscription filters needed

## 4. GuardDuty Automation and Response

### 4.1. EventBridge Integration

GuardDuty publishes findings to **Amazon EventBridge** in near real-time. This is the primary integration pattern for automated response.

**Common automation patterns**:

- **Finding → EventBridge → Lambda**: Add the EC2 instance to a quarantine security group
- **Finding → EventBridge → Step Functions**: Run an incident response workflow
- **Finding → EventBridge → SNS**: Notify the security team
- **Finding → EventBridge → Systems Manager**: Run a remediation document on the instance

### 4.2. Automation Example

1. GuardDuty detects `UnauthorizedAccess:EC2/SSHBruteForce`
2. EventBridge rule matches the finding
3. Lambda function is invoked
4. Lambda creates a new NACL entry to block the offending IP
5. SNS notification is sent to the security team

**Exam scenario**: An EC2 instance is compromised. The security team needs to automatically isolate it → use **GuardDuty → EventBridge → Lambda** to change the instance's security group or create a blocking NACL entry.

### 4.3. GuardDuty + Security Hub

- GuardDuty findings are **automatically forwarded to Security Hub** (if Security Hub is enabled)
- Security Hub provides a single pane of glass for findings from GuardDuty, Inspector, Macie, and other services
- Enables cross-service correlation and consolidated reporting

## 5. GuardDuty Features

### 5.1. Runtime Monitoring

- Monitors OS-level activity on EC2, ECS, and EKS workloads
- Detects: file system access, process execution, network connections, privilege escalation
- Uses a **security agent** deployed on the workload (this is the exception to the agentless model)
- Provides deeper visibility into workload behavior beyond network traffic

### 5.2. Malware Protection

- Scans EBS volumes attached to EC2 instances for malware
- Triggered automatically when a finding suggests potential compromise
- Can generate a finding like `Execution:EC2/MaliciousFile`
- Does not require a separate agent — uses the EC2 instance metadata and EBS snapshots
- Additional cost per GB scanned

### 5.3. RDS Protection

- Monitors RDS login attempts (Amazon Aurora, RDS for MySQL/MariaDB)
- Detects: brute force login attempts, unusual login patterns
- Generates findings for suspicious database authentication activity
- Must be explicitly enabled (not on by default)

### 5.4. Kubernetes Protection (EKS)

- Monitors EKS audit logs for suspicious Kubernetes API activity
- Detects: privilege escalation, anonymous access, suspicious kubectl commands
- Requires EKS audit logging to be enabled

### 5.5. S3 Protection

- Monitors S3 data events for suspicious object-level activity
- Detects: unusual bucket enumeration, data exfiltration patterns
- Must enable S3 protection in GuardDuty settings

## 6. Suppression Rules vs Trusted IP Lists vs Threat Lists

| Feature               | Purpose                                                                     | Scope                       |
| --------------------- | --------------------------------------------------------------------------- | --------------------------- |
| **Suppression rules** | Suppress specific findings that are known false positives                   | Finding-level (by criteria) |
| **Trusted IP lists**  | List of trusted IPs — GuardDuty will NOT generate findings for these IPs    | Account-level               |
| **Threat lists**      | Custom threat intelligence — GuardDuty WILL generate findings for these IPs | Account-level               |

### 6.1. Suppression Rules

- Suppress findings based on **criteria** (finding type, resource, IP, etc.)
- Suppressed findings are still generated but not visible in the console (archived)
- Useful for known false positives (e.g., a security scanning tool that triggers alerts)
- Does NOT prevent GuardDuty from analyzing the underlying data

### 6.2. Trusted IP Lists

- IPs that you trust — GuardDuty will not generate findings for activity from these IPs
- Example: Your corporate VPN egress IPs
- Up to 50 IPs per list
- Applied at the account level

### 6.3. Threat Lists

- IPs you know are malicious — GuardDuty will generate findings for activity from these IPs
- Augments AWS's built-in threat intelligence
- Up to 50,000 IPs per list
- Useful for integrating third-party threat intelligence feeds

## 7. Data Sources and Retention

| Data Source              | GuardDuty Retention                | Notes                                               |
| ------------------------ | ---------------------------------- | --------------------------------------------------- |
| **CloudTrail events**    | 90 days                            | GuardDuty processes but does not store the raw logs |
| **VPC Flow Logs**        | 90 days                            | GuardDuty processes but does not store the raw logs |
| **DNS logs**             | 90 days                            | GuardDuty processes but does not store the raw logs |
| **Finding metadata**     | Retained until archived or expired | Finding history is preserved for investigation      |
| **Malware scan results** | Varies                             | Scan results available during active investigation  |

**Key exam point**: GuardDuty does NOT replace CloudTrail, VPC Flow Logs, or DNS logs — it consumes their data. You should still retain those logs separately.

## 8. GuardDuty Cost Considerations

- Charged based on **volume of data analyzed** (CloudTrail events, VPC Flow Logs data, DNS queries)
- S3 Protection adds additional cost per million events
- Malware Protection charges per GB of EBS volume scanned
- Runtime Monitoring charges per vCPU-hour for monitored workloads
- Costs can be controlled by disabling specific data sources if not needed

## 9. Integration with Other Services

| Service             | Integration                                                              |
| ------------------- | ------------------------------------------------------------------------ |
| **Security Hub**    | Findings automatically forwarded (if enabled)                            |
| **EventBridge**     | Near real-time event-driven automation                                   |
| **Lambda**          | Custom remediation and response                                          |
| **Step Functions**  | Orchestrated incident response workflows                                 |
| **AWS WAF**         | Auto-block malicious IPs via WAF web ACL (via automation)                |
| **Systems Manager** | Run remediation documents on compromised instances                       |
| **Detective**       | Deeper investigation of GuardDuty findings                               |
| **Inspector**       | Complementary — Inspector finds vulnerabilities, GuardDuty finds threats |

## 10. Security Best Practices

- **Enable GuardDuty in all regions and all accounts** — threat actors can operate in any region
- **Designate a delegated administrator** for centralized management
- **Configure automated response** via EventBridge for common finding types
- **Integrate with Security Hub** for centralized finding management
- **Create suppression rules** for known false positives to reduce noise
- **Use trusted IP lists** for known-safe IPs (VPN, partner integrations)
- **Use threat lists** to incorporate third-party threat intelligence
- **Enable S3 Protection, EKS Protection, RDS Protection, Malware Protection** as appropriate
- **Monitor GuardDuty itself** — use CloudTrail to audit `CreateDetector`, `UpdateDetector`, `DeleteDetector` events
- **Export findings to S3** for long-term retention and compliance

## 11. Limits and Quotas

| Resource                         | Limit                        |
| -------------------------------- | ---------------------------- |
| Trusted IP lists per account     | 1                            |
| Threat lists per account         | 1                            |
| IPs per trusted IP list          | 50                           |
| IPs per threat list              | 50,000                       |
| Finding retention                | 90 days (cannot be extended) |
| Detectors per region per account | 1                            |

## 12. GuardDuty vs Other Detection Services

| Service          | What It Detects                        | How                                          |
| ---------------- | -------------------------------------- | -------------------------------------------- |
| **GuardDuty**    | Malicious activity and threats         | ML + threat intelligence + anomaly detection |
| **Inspector**    | Vulnerabilities in EC2, ECR, Lambda    | Network reachability + CVEs                  |
| **Macie**        | Sensitive data in S3                   | ML-based data classification                 |
| **Detective**    | Root cause analysis of security events | Pre-computed data + graph analysis           |
| **Security Hub** | Aggregated findings and compliance     | Collects findings from multiple services     |

## 13. Remediation Options

| Finding Type                                                               | Possible Automated Response                              |
| -------------------------------------------------------------------------- | -------------------------------------------------------- |
| Malicious IP calling APIs (`UnauthorizedAccess:IAMUser/MaliciousIPCaller`) | Create SCP or IAM policy to deny access from that IP     |
| EC2 C2 communication (`Backdoor:EC2/C&CActivity.B`)                        | Isolate the instance (change SG), take forensic snapshot |
| SSH brute force (`UnauthorizedAccess:EC2/SSHBruteForce`)                   | Block the source IP via NACL or WAF                      |
| IAM credential misuse                                                      | Rotate credentials immediately, revoke sessions          |
| RDS brute force                                                            | Enable RDS Proxy, enforce strong auth, block source IP   |

## 14. Exam Tips

1. **Agentless by default** — GuardDuty consumes CloudTrail, VPC Flow Logs, and DNS logs without agents. Runtime Monitoring is the exception (requires an agent).

2. **Multi-account** — use the **delegated administrator** pattern for central management across AWS Organizations.

3. **EventBridge** is the primary integration for automated response — GuardDuty → EventBridge → Lambda/Step Functions.

4. **Security Hub** automatically receives GuardDuty findings — the two services work together.

5. **Suppression rules** handle false positives; **trusted IP lists** whitelist IPs; **threat lists** add custom threat intel.

6. **90-day finding retention** — findings are not retained indefinitely. Export findings to S3 or Security Hub for longer retention.

7. **S3 Protection** detects data exfiltration but must be explicitly enabled.

8. **Malware Protection** scans EBS volumes — triggered automatically on suspected compromise.

9. **RDS Protection** monitors database login attempts — also must be explicitly enabled.

10. **GuardDuty cost** is based on data volume analyzed (CloudTrail events, flow logs, DNS queries).

11. **GuardDuty does not replace CloudTrail** — it consumes CloudTrail data. You still need CloudTrail for long-term auditing and compliance.

12. **Delegated admin** can enable GuardDuty for all member accounts. Member accounts cannot disable it.

13. **Finding format**: `<ThreatPurpose>:<ResourceType>/<ThreatName>` — e.g., `Backdoor:EC2/C&CActivity.B`, `UnauthorizedAccess:IAMUser/MaliciousIPCaller`.

14. **Inspector vs GuardDuty**: Inspector finds vulnerabilities (CVEs, network reachability). GuardDuty finds active threats (malicious activity, compromise). They are complementary.

15. **Detective** is used for deeper investigation after GuardDuty finds something suspicious — GuardDuty alerts, Detective investigates.
