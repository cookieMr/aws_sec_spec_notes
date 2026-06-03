# Amazon Macie

<figure>
  <img src="./images/Arch_Amazon-Macie_64@5x.png" alt="Amazon Macie Icon" width=200>
  <figcaption><center>Amazon Macie<br><i>Image source: AWS Documentation</i></center></figcaption>
</figure>

**Overview**: Amazon Macie is a fully managed data security and data privacy service that uses machine learning and pattern matching to discover and protect sensitive data stored in Amazon S3. Macie identifies PII, PHI, financial data, and credentials, and monitors S3 bucket policies for security risks.

**Domain weight**: Macie is the primary S3 data discovery and classification service in the Data Protection domain (~18% of SCS-C03) and appears in questions about sensitive data governance, compliance (HIPAA, PCI DSS, GDPR), and data exfiltration prevention.

## 1. What Macie Does

| Feature                                | Description                                                       |
| -------------------------------------- | ----------------------------------------------------------------- |
| **S3 bucket inventory**                | Automatically inventories all S3 buckets in the account           |
| **Automated sensitive data discovery** | Scans S3 buckets for sensitive data using ML and pattern matching |
| **Managed data identifiers**           | Built-in detection for PII, PHI, credentials, financial data      |
| **Custom data identifiers**            | User-defined patterns for proprietary sensitive data              |
| **Policy monitoring**                  | Detects S3 bucket policy changes that could expose data           |
| **Findings**                           | Generates findings for sensitive data and security risks          |
| **Integration**                        | Sends findings to Security Hub, EventBridge                       |

### 1.1. Macie v2 (Current)

Macie v2 is the current generation. It focuses exclusively on **S3 data security** — it does not scan databases, EBS volumes, or other data stores.

## 2. Sensitive Data Discovery

### 2.1. Managed Data Identifiers

Macie includes a library of managed data identifiers that recognize common sensitive data types:

| Category        | Examples                                                                                          |
| --------------- | ------------------------------------------------------------------------------------------------- |
| **PII**         | Names, email addresses, phone numbers, mailing addresses, SSN, driver's license, passport numbers |
| **PHI**         | Health insurance numbers, medical record numbers                                                  |
| **Credentials** | AWS secret keys, database passwords, API keys, private keys                                       |
| **Financial**   | Credit card numbers, bank account numbers, routing numbers                                        |
| **Other**       | Tax identification numbers (EIN), vehicle identification numbers (VIN)                            |

### 2.2. Custom Data Identifiers

- Define your own patterns using **regular expressions** and **keyword lists**
- Example: Customer account numbers, employee IDs, proprietary codes
- Can include context words to reduce false positives (e.g., require "SSN" near a 9-digit number)
- Maximum: 20 custom data identifiers per account

**Exam scenario**: A company needs to detect proprietary customer account numbers stored in S3 → create a **custom data identifier** with the account number regex and relevant keywords.

### 2.3. Sensitive Data Discovery Jobs

- **Automated (default)**: Macie continuously samples and analyzes objects in S3 buckets
- **One-time job**: Run a targeted scan of specific buckets or objects
- **Scheduled job**: Run scans at specified intervals
- Sampling rate is configurable — scanning more objects increases cost but improves coverage
- Jobs can analyze both the **bucket policy** and **object content**

**Exam tip**: Macie does NOT scan every object by default — it uses **sampling**. For comprehensive coverage, configure the sampling percentage or run a full one-time job.

## 3. S3 Bucket Monitoring

Macie provides an S3 bucket inventory and monitors each bucket for:

| Monitoring Type                   | What It Detects                                                                |
| --------------------------------- | ------------------------------------------------------------------------------ |
| **Bucket-level permissions**      | Public read/write access, bucket policies granting access to external accounts |
| **Encryption status**             | Buckets without default encryption enabled                                     |
| **Replication status**            | Buckets or objects replicated outside the account                              |
| **Object-level sensitive data**   | Objects containing PII, PHI, credentials, etc.                                 |
| **Shared with AWS Organizations** | Buckets accessible to accounts outside the organization                        |

- Macie provides a **bucket-level security score** for each bucket
- The dashboard shows overall data security posture across all S3 buckets

## 4. Findings

Macie generates two types of findings:

### 4.1. Sensitive Data Findings

| Severity     | Example                                                                     |
| ------------ | --------------------------------------------------------------------------- |
| **Critical** | Large volume of SSNs or credit card numbers in a publicly accessible bucket |
| **High**     | PII in a bucket shared with external accounts                               |
| **Medium**   | Credentials or API keys in a bucket                                         |
| **Low**      | Minor policy deviations                                                     |

### 4.2. Policy Findings

| Finding                                            | Description                                                 |
| -------------------------------------------------- | ----------------------------------------------------------- |
| `Policy:IAMUser/S3BucketPublic`                    | Bucket is publicly accessible                               |
| `Policy:IAMUser/S3BucketSharedWithExternalAccount` | Bucket policy grants access to external AWS accounts        |
| `Policy:IAMUser/S3BucketReplicatedExternally`      | Objects in the bucket are replicated to an external account |
| `Policy:IAMUser/S3BucketEncryptionDisabled`        | Bucket does not have default encryption enabled             |

### 4.3. Suppression Rules

- Findings can be suppressed based on criteria (finding type, bucket, etc.)
- Suppressed findings are archived but still available for audit
- Useful for known or accepted risks

## 5. Integration with Other Services

| Service               | Integration                                                          |
| --------------------- | -------------------------------------------------------------------- |
| **Security Hub**      | All Macie findings are automatically forwarded (if enabled)          |
| **EventBridge**       | Findings published as events for automated response                  |
| **Lambda**            | Custom remediation actions (e.g., apply bucket policy, move objects) |
| **Step Functions**    | Orchestrated incident response workflows                             |
| **AWS Organizations** | Delegated administrator for multi-account management                 |

### 5.1. Macie + Security Hub

- Findings appear automatically in Security Hub
- Enables cross-service correlation (GuardDuty + Macie + Inspector)
- Provides a single pane of glass for all security findings

### 5.2. Macie + EventBridge Automation

Common automation patterns:

- **Finding → EventBridge → Lambda**: Automatically apply bucket policy to block public access
- **Finding → EventBridge → SNS**: Notify the data owner
- **Finding → EventBridge → Step Functions**: Run a response workflow (notify owner, move data to quarantined bucket, apply encryption)

**Exam scenario**: Sensitive data is detected in a publicly accessible S3 bucket → **Macie → EventBridge → Lambda** to automatically apply a deny-all-except-owner bucket policy.

## 6. Multi-Account Management

- Designate a **delegated administrator** from the management account of AWS Organizations
- The delegated admin can enable Macie for all member accounts automatically
- Central view of all S3 bucket inventories and findings across accounts
- Member accounts can manage their own Macie settings if not restricted

**Exam scenario**: A security team needs to monitor for sensitive data across 100 AWS accounts → designate a **Macie delegated administrator** for organization-wide data discovery.

## 7. Cost

| Cost Driver                 | Details                                            |
| --------------------------- | -------------------------------------------------- |
| **S3 bucket inventory**     | Free — basic bucket metadata monitoring is no-cost |
| **Automated discovery**     | Charged per GB of data processed (sampled objects) |
| **Custom data identifiers** | Included in the automated discovery cost           |
| **One-time/scheduled jobs** | Charged per GB of data scanned                     |

- Cost depends on the volume of data sampled and the sampling rate
- Higher sampling rates = more accurate results = higher cost
- Can be reduced by excluding non-critical buckets from automated discovery

## 8. Security Best Practices

- **Enable Macie in all accounts** — sensitive data can exist anywhere
- **Use automated discovery** for continuous monitoring
- **Configure custom data identifiers** for organization-specific sensitive data
- **Integrate with Security Hub** for centralized finding management
- **Set up EventBridge automation** for rapid response to sensitive data exposure
- **Review findings regularly** and verify false positives vs true positives
- **Use suppression rules** for known acceptable risks
- **Apply S3 Block Public Access** at the account level as a preventive control — Macie detects, Block Public Access prevents

## 9. Limits and Quotas

| Resource                              | Limit               |
| ------------------------------------- | ------------------- |
| Custom data identifiers per account   | 20                  |
| Sensitivity data discovery jobs       | 100 concurrent jobs |
| Maximum objects per S3 bucket scanned | Unlimited           |
| Finding retention                     | 90 days (in Macie)  |

## 10. Macie vs Other Services

| Service          | What It Detects                                         |
| ---------------- | ------------------------------------------------------- |
| **Macie**        | Sensitive data in S3 (PII, PHI, credentials, financial) |
| **GuardDuty**    | Active threats and malicious activity                   |
| **Inspector**    | Vulnerabilities (CVEs, network exposure)                |
| **Config**       | Resource configuration compliance                       |
| **Security Hub** | Aggregated findings from all services                   |

**Exam tip**: Macie is the **only** AWS service that discovers sensitive data content in S3 objects. If the question involves PII, PHI, credit cards, or credentials stored in S3, the answer is Macie.

## 11. Macie vs S3 Access Logs

| Feature                  | Macie                      | S3 Access Logs          |
| ------------------------ | -------------------------- | ----------------------- |
| Content analysis         | Yes (scans object content) | No                      |
| Access monitoring        | Policy-level only          | Yes (who accessed what) |
| Sensitive data detection | Yes                        | No                      |
| Real-time                | Near-real-time             | Delayed (hours)         |
| Automated response       | Via EventBridge            | Via EventBridge         |

## 12. Exam Tips

1. **Macie is for S3 only** — it does not scan RDS, DynamoDB, EBS, or other data stores. If a question involves sensitive data in S3, think Macie.

2. **Managed data identifiers** detect common sensitive data types (PII, PHI, credentials, financial data). **Custom data identifiers** use regex for proprietary data.

3. **Automated discovery** uses sampling — not every object is scanned. For a full audit, run a one-time job.

4. **Two finding types**: **Sensitive data findings** (content contains PII/etc.) and **Policy findings** (bucket is public, shared externally, unencrypted).

5. **Security Hub integration** is automatic — Macie findings appear in Security Hub without additional configuration.

6. **EventBridge integration** enables automated remediation — e.g., auto-block public access when sensitive data is detected.

7. **Multi-account**: Use a delegated administrator for organization-wide sensitive data discovery.

8. **Macie does not prevent data exposure** — it detects and alerts. Use S3 Block Public Access, SCPs, and IAM policies for prevention.

9. **Macie v2 is S3-focused**. Macie Classic (previous version) also monitored CloudTrail and other sources — Macie v2 is exclusively S3.

10. **Suppression rules** handle false positives — they archive findings based on criteria.

11. **Bucket security score** helps prioritize which buckets need immediate attention.

12. **Compliance** (HIPAA, PCI DSS, GDPR): Macie helps identify where sensitive data is stored for compliance reporting. It does not enforce compliance — it discovers where sensitive data resides so you can take action.

13. **Cost management**: Exclude low-value buckets from automated discovery and adjust the sampling percentage to control costs.

14. **Sampling is configurable**: By default, Macie samples objects. Increase the sampling percentage for higher-risk buckets, decrease for low-risk buckets.

15. **Macie + GuardDuty + Inspector**: Macie finds sensitive data, GuardDuty detects active threats, Inspector finds vulnerabilities — together they provide comprehensive security coverage.
