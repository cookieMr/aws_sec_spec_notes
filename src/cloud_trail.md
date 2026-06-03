# AWS CloudTrail

<figure>
  <img src="./images/Arch_AWS-CloudTrail_64@5x.png" alt="AWS CloudTrail Icon" width=200>
  <figcaption><center>AWS CloudTrail<br><i>Image source: AWS Documentation</i></center></figcaption>
</figure>

**Overview**: AWS CloudTrail records every API call made in your AWS account — who made it, when, from where, what action, and what the result was. It is the foundation of AWS security auditing, logging, and monitoring. Nearly every exam scenario involving detective controls, incident response, or compliance auditing involves CloudTrail.

**Domain weight**: CloudTrail is the most important service in the Security Logging and Monitoring domain (~18% of SCS-C03) and appears across all domains — it is the single most tested security service on the exam alongside IAM and KMS.

## 1. What CloudTrail Records

CloudTrail captures all API calls — programmatic (SDK, CLI), console actions, and AWS service-to-service calls. Every event includes:

| Key Field            | Description                                                            |
| -------------------- | ---------------------------------------------------------------------- |
| `eventName`          | The API action performed (e.g., `RunInstances`, `CreateUser`)          |
| `eventSource`        | The service the action was performed on (e.g., `ec2.amazonaws.com`)    |
| `userIdentity`       | Who made the call (IAM user, role, federated user, root, assumed role) |
| `sourceIPAddress`    | The IP address the call came from                                      |
| `eventTime`          | When the call was made (UTC)                                           |
| `requestParameters`  | The input parameters of the API call                                   |
| `responseElements`   | The output of the API call                                             |
| `errorCode`          | If the call failed, the error code (e.g., `AccessDenied`)              |
| `errorMessage`       | If the call failed, a human-readable error message                     |
| `resources`          | List of resources involved in the API call                             |
| `recipientAccountId` | The account that owns the resource being accessed                      |

## 2. Event Types

| Event Type            | What It Logs                                                                                                      | Default Logging        | Cost                        |
| --------------------- | ----------------------------------------------------------------------------------------------------------------- | ---------------------- | --------------------------- |
| **Management events** | Control plane operations (e.g., `CreateVPC`, `AttachRolePolicy`, `RunInstances`)                                  | Enabled by default     | Free (90-day Event History) |
| **Data events**       | Data plane operations on specific resources (e.g., `GetObject` on S3, `Invoke` on Lambda, `PutItem` on DynamoDB)  | Not enabled by default | Extra cost                  |
| **Insights events**   | Unusual API activity detected by CloudTrail (e.g., spikes in `TerminateInstances`, unusual `AssumeRole` patterns) | Not enabled by default | Extra cost                  |

### 2.1. Management Events

- Logged **by default** for every AWS account
- Free — accessible via Event History for 90 days
- Two sub-types:
  - **Read events**: Operations that read/list resources (e.g., `DescribeInstances`, `ListBuckets`)
  - **Write events**: Operations that create, modify, or delete resources (e.g., `CreateUser`, `DeleteBucket`)
- Can be separated into read-only and write-only trails

### 2.2. Data Events

- **Not logged by default** — you must explicitly enable them on a trail
- High volume, so **extra cost** applies
- Common data events on the exam:
  - S3 object-level operations (`GetObject`, `PutObject`, `DeleteObject`)
  - Lambda function invocations (`Invoke`)
  - DynamoDB item operations (`GetItem`, `PutItem`, `DeleteItem`)
  - SQS queue operations (`SendMessage`, `ReceiveMessage`)
- S3 data events can be scoped to specific buckets using event selectors
- **Exam scenario**: A security team needs to detect unauthorized access to objects in a specific S3 bucket → enable **S3 data events** on that bucket in a CloudTrail trail.

### 2.3. Insights Events

- CloudTrail continuously monitors management event volume
- When activity deviates from normal patterns (unusual spikes), CloudTrail logs an Insights event
- Example: A sudden spike in `TerminateInstances` or `DeleteLogGroup` calls triggers an Insights event
- Useful for detecting potential compromise early
- Must be enabled on a trail — not free

## 3. Event History vs Trails vs CloudTrail Lake

| Feature           | Event History       | Trails                                       | CloudTrail Lake               |
| ----------------- | ------------------- | -------------------------------------------- | ----------------------------- |
| Retention         | 90 days             | As long as you keep the S3 logs              | Configurable (up to 10 years) |
| Cost              | Free                | S3 storage costs + optional CloudWatch costs | Per-ingestion and per-storage |
| Data events       | No                  | Yes                                          | Yes                           |
| Insights          | No                  | Yes                                          | Yes                           |
| Multi-region      | No                  | Yes                                          | Yes                           |
| Organization      | No                  | Yes                                          | Yes                           |
| Query capability  | Console search only | Athena on S3 or external tools               | Built-in SQL query            |
| Long-term storage | No                  | S3 (indefinite)                              | Indigenous storage            |

### 3.1. Event History

- Free, automatically enabled — viewable in the CloudTrail console
- 90-day rolling window
- Only management events
- Cannot be customized — no S3 delivery, no data events, no Insights
- Useful for quick lookups but not for long-term retention or compliance

### 3.2. Trails

- A "trail" delivers events to an S3 bucket (and optionally to CloudWatch Logs / EventBridge)
- Key configuration choices:

| Setting             | Options                                         | Notes                                           |
| ------------------- | ----------------------------------------------- | ----------------------------------------------- |
| Scope               | **Single-region** or **Multi-region**           | Multi-region trails log events from all regions |
| Events              | Management, Data, or both                       | Data events must be explicitly enabled          |
| Delivery            | S3 only, S3 + CloudWatch Logs, S3 + EventBridge | CloudWatch Logs adds cost                       |
| Encryption          | SSE-S3 (default) or SSE-KMS                     | SSE-KMS adds cost for KMS API calls             |
| Log file validation | Enabled or disabled                             | Recommended — enables integrity verification    |
| Organization trail  | All accounts in AWS Organizations               | Only the management account can create          |

- **Multi-region trail**: A single trail that logs events from every AWS region. Events are delivered to a single S3 bucket. When a new region is added, CloudTrail automatically starts logging it.
- **Organization trail**: Created in the management account of AWS Organizations. Automatically logs all accounts in the organization. All accounts can see the trail in their CloudTrail console but cannot modify or delete it.

### 3.3. CloudTrail Lake

- Managed data lake for CloudTrail data
- Supports SQL-based queries directly
- Retention configurable up to 10 years
- **Note**: As of May 31, 2026, CloudTrail Lake is **closed to new customers** — existing customers can continue using it.
- If the exam asks about long-term, queryable CloudTrail storage, the answer is typically **S3 + Athena** (now that Lake is closed to new customers)

## 4. Log File Integrity

### 4.1. Log File Integrity Validation

- When enabled, CloudTrail creates a **digest file** every hour that contains hashes of the log files
- The digest files are signed using a private key — only AWS has the key
- You can verify the digest files using the public key to confirm:
  1. The log files have not been tampered with
  2. No log files were deleted
  3. The log files were created by CloudTrail (not forged)
- Digest files are stored in the same S3 bucket as the logs (under a separate `AWSLogs/<account>/CloudTrail-Digest/` prefix)
- Enabling this is an **exam best practice** — it ensures log integrity and non-repudiation

**Exam scenario**: The security team needs to prove CloudTrail logs have not been tampered with since creation → enable **log file integrity validation** and periodically validate the digest files.

### 4.2. Integrity Validation Tools

- AWS CLI: `aws cloudtrail validate-logs`
- The CLI command automatically downloads the digest files, verifies hashes, and checks signatures
- Can be run on a schedule (e.g., daily Lambda function) to automate validation

## 5. Log Delivery Destinations

| Destination         | Purpose                                          | Exam Notes                     |
| ------------------- | ------------------------------------------------ | ------------------------------ |
| **S3 bucket**       | Long-term storage, archival, compliance          | Primary destination for trails |
| **CloudWatch Logs** | Real-time monitoring, metric filters, alarms     | Enables real-time alerting     |
| **EventBridge**     | Event-driven automation, cross-account streaming | For automated responses        |

### 5.1. S3 Delivery

- Default destination for all trails
- Log file format: JSON, compressed (gzip)
- Log files are delivered approximately every 5 minutes
- S3 path pattern: `bucket/AWSLogs/<account>/CloudTrail/<region>/<year>/<month>/<day>/`
- **Best practice**: Enable S3 bucket **versioning** to protect against accidental deletion or overwrite
- **Best practice**: Use a bucket policy to prevent deletion of logs (e.g., `s3:DeleteObject` Deny for all principals except a break-glass admin)
- **Best practice**: Use **S3 Object Lock** to make logs immutable (WORM mode — Write Once, Read Many)

### 5.2. CloudWatch Logs Delivery

- Enables **real-time monitoring** via metric filters and alarms
- Example metric filter: Count the number of `AccessDenied` events or `UnauthorizedOperation` errors
- Example alarm: Trigger an SNS notification when more than 5 `CreateUser` calls occur in 5 minutes
- **Cost**: You pay for CloudWatch Logs ingestion and storage
- A **service-linked role** (`CloudTrail_CloudWatchLogs_Role`) is created to grant CloudTrail permission to deliver logs to CloudWatch

### 5.3. EventBridge Delivery

- Enables event-driven automation
- Example: Send an event when `CreateNetworkAclEntry` is called, triggering a Lambda function to verify the NACL entry against security policy
- Supports cross-account event streaming (to a central security account)
- Useful for Security Hub and GuardDuty automation

## 6. Encryption

| Option      | Description                                                 | Cost      |
| ----------- | ----------------------------------------------------------- | --------- |
| **SSE-S3**  | Default encryption — AES-256 managed by S3                  | Free      |
| **SSE-KMS** | Customer-managed KMS key — you control who can decrypt logs | KMS costs |
| **SSE-C**   | Not supported for CloudTrail logs                           | N/A       |

- If using SSE-KMS:
  - You need a KMS key policy that allows CloudTrail to use the key
  - The trail must include the KMS key ID
  - You control decryption access separately from S3 access (defense in depth)
  - **Caution**: A misconfigured KMS key policy can break log delivery

**Exam scenario**: You need to ensure only the security team can decrypt CloudTrail logs, while the operations team can access the S3 bucket but not read log contents → use **SSE-KMS** and restrict the KMS key decrypt permissions to the security team.

## 7. Organization Trails

- Created in the management account of AWS Organizations
- Logs all API activity across **all accounts** in the organization
- All member accounts automatically see the organization trail in their console but **cannot modify or delete it**
- Member accounts do not pay for the organization trail — the management account pays
- An organization trail **must** be a multi-region trail
- **Exam scenario**: A security team needs centralized logging across 50 AWS accounts → create an **organization trail** in the management account delivering to a central S3 bucket.

## 8. CloudTrail and S3

### 8.1. Key S3 Bucket Settings for CloudTrail

| Setting                                | Benefit                                                      |
| -------------------------------------- | ------------------------------------------------------------ |
| **Bucket versioning**                  | Protects logs from accidental deletion or overwrite          |
| **S3 Object Lock**                     | Immutable log storage — prevents deletion even by root users |
| **MFA Delete**                         | Requires MFA to delete objects — highly secure               |
| **Bucket policy denying DeleteObject** | Prevents log deletion by most principals                     |
| **SSE-KMS encryption**                 | Defense-in-depth — separates log access from log decryption  |
| **Lifecycle rules**                    | Transition to Glacier after 90 days for cost savings         |
| **S3 Access Logs**                     | Audit access to the CloudTrail S3 bucket itself              |
| **Block Public Access**                | Critical — never expose CloudTrail logs publicly             |

### 8.2. S3 Bucket Policy for CloudTrail

- CloudTrail requires a specific bucket policy to write logs
- The policy grants `s3:PutObject` to the CloudTrail service principal
- Includes a condition that the objects must use `bucket-owner-full-control` ACL
- Exam may ask: "Why are CloudTrail logs not being delivered?" → check the **bucket policy** or **KMS key policy** (if SSE-KMS is used)

## 9. CloudTrail vs Other Logging Services

| Service             | What It Logs                   | Logs User Actions? | Logs API Calls? | Logs Network Traffic? |
| ------------------- | ------------------------------ | ------------------ | --------------- | --------------------- |
| **CloudTrail**      | API activity (who, what, when) | Yes                | Yes             | No                    |
| **VPC Flow Logs**   | Network traffic metadata       | Partial (IP only)  | No              | Yes                   |
| **S3 Access Logs**  | S3 object access requests      | Yes                | Yes (S3 only)   | No                    |
| **CloudWatch Logs** | Application and system logs    | Varies             | Varies          | No                    |
| **AWS Config**      | Resource configuration changes | Yes                | No              | No                    |

**Exam tip**: If the question asks about who performed a specific API action and when → **CloudTrail**. If it asks about network traffic patterns → **VPC Flow Logs**. If it asks about resource configuration changes over time → **AWS Config**.

## 10. CloudTrail and Global Services

- Some AWS services are **global** (e.g., IAM, STS, Route 53, CloudFront, WAF)
- Global service events are logged in **us-east-1** (the default global endpoint)
- If you create a single-region trail in a specific region, it will NOT capture global service events
- To capture global service events, you must either:
  1. Create a **multi-region trail** (recommended), OR
  2. Create a trail in **us-east-1**
- **Exam scenario**: An administrator notices that IAM `CreateUser` calls are not appearing in CloudTrail logs → the trail may be **single-region** and not logging global service events from us-east-1.

## 11. Monitoring CloudTrail Itself

- CloudTrail API calls are logged in CloudTrail (meta-logging)
- You can monitor for:
  - `StopLogging` — someone disabled CloudTrail
  - `UpdateTrail` — someone modified trail settings
  - `DeleteTrail` — someone deleted the trail
  - `PutEventSelectors` — someone changed what events are logged
- **Critical**: Set up a CloudWatch alarm on the trail's `StopLogging` event — this is a common first action for attackers covering their tracks

## 12. Best Practices

### 12.1. Trail Configuration

- **Always use a multi-region trail** — ensures no region is missed
- **Use an organization trail** in AWS Organizations environments
- **Enable data events** for critical S3 buckets (especially those containing sensitive data)
- **Enable Insights events** for early threat detection
- **Enable log file integrity validation** for tamper-proofing
- **Use SSE-KMS** for defense-in-depth encryption

### 12.2. S3 Bucket Configuration

- Lock down the CloudTrail S3 bucket with **S3 Object Lock** (WORM)
- Enable **bucket versioning**
- Apply **deny DeleteObject** bucket policy
- Enable **S3 Access Logs** for the bucket to audit access to the trail bucket
- Use **lifecycle policies** to transition older logs to lower-cost storage (e.g., Glacier after 90 days)
- **Never** — under any circumstance — make the CloudTrail S3 bucket public

### 12.3. Monitoring and Alerting

- Set up CloudWatch metric filters and alarms for:
  - `StopLogging` events
  - `DeleteTrail` events
  - `UpdateTrail` events
  - `UnauthorizedOperation` errors
  - `AccessDenied` errors (potential attack)
- Integrate with **Security Hub** for centralized finding aggregation
- Forward to **central security account** via EventBridge for multi-account environments

### 12.4. Access Control

- Separate write access (who can create/modify trails) from read access (who can view logs)
- Use IAM policies to restrict which users can stop, delete, or modify trails
- Use SSE-KMS to separate log access from log decryption

## 13. Limits and Quotas

| Resource                                  | Limit               |
| ----------------------------------------- | ------------------- |
| Trails per account (multi-region)         | 5                   |
| Trails per account (single-region)        | 5 per region        |
| Event History retention                   | 90 days             |
| S3 delivery frequency                     | ~5 minute intervals |
| CloudTrail Lake retention (if still used) | Up to 10 years      |
| Max S3 bucket lifecycle rules             | 1,000               |

## 14. Common Troubleshooting Scenarios

### 14.1. Logs Not Being Delivered to S3

1. Check the **S3 bucket policy** — CloudTrail requires specific permissions to write logs
2. If using **SSE-KMS**, check the **KMS key policy** — CloudTrail must have `kms:GenerateDataKey` and `kms:Decrypt` permissions
3. Check that the S3 bucket is in the **same region** as the trail (or that the trail is multi-region)
4. Verify the trail is **not stopped** (`IsLogging = false`)

### 14.2. Global Service Events Missing

- The trail must be **multi-region** or specifically logging from **us-east-1**
- Single-region trails outside us-east-1 miss IAM, STS, Route 53, CloudFront, and WAF events

### 14.3. Data Events Not Appearing

- Data events are **not enabled by default** — they must be explicitly selected when creating/updating the trail
- S3 data events require specifying the bucket ARN (or using a selector template)
- Data events incur **additional cost**

## 15. Exam Tips

1. **CloudTrail is always the first place to look** in incident response questions about API activity. If a question asks "how to determine who deleted an S3 bucket", the answer is CloudTrail.

2. **Management events are free and on by default**. Data events and Insights events must be explicitly enabled and cost extra.

3. **Multi-region trails** are the standard. Single-region trails are only used in specific scenarios. If you see a question about global services (IAM, STS, CloudFront), the trail must capture us-east-1 events.

4. **Organization trails** log activity across all accounts in AWS Organizations. Only the management account can create/delete them.

5. **Log file integrity validation** proves logs have not been tampered with. Use `aws cloudtrail validate-logs` to verify.

6. **SSE-KMS** separates log access from log decryption — the security team can control the KMS key while operations access the S3 bucket.

7. **CloudWatch Logs integration** enables real-time monitoring. Use metric filters to detect suspicious patterns.

8. **EventBridge** enables automated response actions based on CloudTrail events.

9. **S3 Object Lock** makes CloudTrail logs immutable (WORM). This is critical for compliance (SEC, FINRA, HIPAA).

10. **Monitor CloudTrail itself** — attackers often stop or delete trails to cover their tracks. Set up `StopLogging` alerts.

11. **90-day Event History** is free but cannot be extended. For longer retention, create a trail.

12. **Data events are high volume** — only enable them for specific resources you need to monitor, not all resources.

13. **Insights events** detect anomalous API patterns — useful for early compromise detection.

14. **CloudTrail Lake** (now closed to new customers as of May 2026) — existing users can continue using it. For new deployments, use **S3 + Athena** for queryable long-term log storage.

15. **IAM permissions** to create/manage trails: `cloudtrail:CreateTrail`, `cloudtrail:UpdateTrail`, `cloudtrail:DeleteTrail`, `cloudtrail:StartLogging`, `cloudtrail:StopLogging`. Restrict these tightly.
