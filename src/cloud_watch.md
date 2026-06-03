# Amazon CloudWatch & CloudWatch Logs

<figure>
  <img src="./images/Arch_Amazon-CloudWatch_64@5x.png" alt="Amazon CloudWatch Icon" width=200>
  <figcaption><center>Amazon CloudWatch<br><i>Image source: AWS Documentation</i></center></figcaption>
</figure>

**Overview**: Amazon CloudWatch is AWS's monitoring and observability service. For the Security Specialty exam, CloudWatch Logs is the most relevant component — it is the primary destination for real-time log analysis, metric filters, and alarms based on log content. CloudWatch is where CloudTrail events, VPC Flow Logs, and application logs converge for monitoring and alerting.

**Domain weight**: CloudWatch is central to the Security Logging and Monitoring domain (~18% of SCS-C03) and appears in nearly every incident response, alerting, and compliance scenario.

## 1. CloudWatch Logs Core Concepts

| Concept                 | Description                                                                                  |
| ----------------------- | -------------------------------------------------------------------------------------------- |
| **Log event**           | A single record of activity (timestamp + raw message)                                        |
| **Log stream**          | A sequence of log events from the same source (e.g., one EC2 instance, one Lambda container) |
| **Log group**           | A container for log streams with shared settings (retention, encryption, metric filters)     |
| **Metric filter**       | A pattern-matching rule that extracts metric data from log events                            |
| **Subscription filter** | Real-time streaming of log events to Lambda, Kinesis, or Firehose                            |

### 1.1. Log Retention

- Default: **Never expire** (logs are kept indefinitely)
- Configurable per log group: 1 day to 10 years
- Reducing retention deletes older logs immediately (cannot be recovered)
- Retention policies are part of compliance — ensure logs are kept for required audit periods
- **Exam scenario**: Logs must be retained for 7 years for regulatory compliance → set log group retention to **10 years** (max is 10 years) or export to S3 for longer-term archival

### 1.2. Encryption

| Option         | Description                                                                    |
| -------------- | ------------------------------------------------------------------------------ |
| **Default**    | Logs are encrypted at rest using AWS-managed keys (no control over decryption) |
| **KMS**        | Encrypt log group data with a customer-managed KMS key                         |
| **In transit** | TLS encryption for log delivery and API access                                 |

- KMS encryption can be enabled at log group creation time
- Once KMS is associated with a log group, **all future data** is encrypted with that key
- Decryption is transparent to authorized users (CloudWatch decrypts on read)
- KMS key policy must grant CloudWatch Logs permission to use the key

## 2. Metric Filters

### 2.1. How They Work

- Define a **pattern** to match against incoming log events
- When a log event matches the pattern, CloudWatch increments a metric
- Metrics are published to CloudWatch as custom metrics
- Alarms can be triggered on these metrics
- Metric filters are defined at the **log group** level

### 2.2. Common Security Metric Filters

| Filter Pattern                                               | What It Detects                    | Use Case                                         |
| ------------------------------------------------------------ | ---------------------------------- | ------------------------------------------------ |
| `"AccessDenied"`                                             | Denied API calls                   | Potential reconnaissance or privilege escalation |
| `"UnauthorizedOperation"`                                    | Unauthorized API calls             | Security policy violations                       |
| `"root"` and `"userIdentity.type":"Root"`                    | Root user activity                 | Monitor root account usage                       |
| `"ConsoleLogin"` and `"sourceIPAddress"` not in allowed list | Console logins from unexpected IPs | Unauthorized access detection                    |
| `"StopLogging"`                                              | CloudTrail being disabled          | Attacker covering tracks (critical)              |
| `"DeleteTrail"`                                              | CloudTrail deletion                | Attacker covering tracks (critical)              |
| `"CreateUser"` or `"CreateAccessKey"`                        | IAM user/key creation              | Potential backdoor creation                      |
| `"AuthorizeSecurityGroupIngress"`                            | Security group rule changes        | Network perimeter changes                        |
| `"FailedToAuthenticate"`                                     | Failed authentication              | Brute force attempts                             |
| `"PasswordChanged"` or `"UpdateLoginProfile"`                | IAM password changes               | Account compromise indicator                     |

### 2.3. Metric Filter Example

```
[,,, "AccessDenied", ...]
```

This pattern matches any log event that contains the string `AccessDenied`. The metric is incremented by 1 for each matching event. You can also extract numeric values and dimensions from log events using more complex patterns.

### 2.4. Alarm Actions

When a metric filter's alarm triggers, it can perform:

| Action           | Destination              | Use Case                                    |
| ---------------- | ------------------------ | ------------------------------------------- |
| **SNS**          | Email, SMS, HTTP, Lambda | Notify security team                        |
| **Auto Scaling** | Scaling policy           | Scale out in response to high error rates   |
| **EC2 action**   | Stop, terminate, reboot  | Automatic response to compromised instances |

## 3. CloudWatch Alarms

### 3.1. Alarm States

| State                 | Meaning                            |
| --------------------- | ---------------------------------- |
| **OK**                | Metric is within the threshold     |
| **ALARM**             | Metric has breached the threshold  |
| **INSUFFICIENT_DATA** | Not enough data to determine state |

### 3.2. Alarm Configuration

- Based on a CloudWatch metric (from any source: custom, AWS service, metric filter)
- Threshold: Static (e.g., `> 5`) or Anomaly Detection (ML-based)
- Period: How often the metric is evaluated (60s to 1 day)
- Evaluation periods: How many consecutive periods must breach before ALARM
- Datapoints to alarm: How many of those periods must breach

**Exam scenario**: An alarm that fires on a single data point will trigger more false positives than one that requires 3 consecutive breaching periods. Know how to distinguish between sensitivity and false positive reduction.

## 4. Subscription Filters

### 4.1. Purpose

- Real-time streaming of log events from a log group to a destination
- Unlike metric filters (which only create metrics), subscription filters send the raw log events

### 4.2. Destinations

| Destination               | Best For                                                           |
| ------------------------- | ------------------------------------------------------------------ |
| **Lambda**                | Custom processing, enrichment, alerting                            |
| **Kinesis Data Streams**  | High-throughput streaming to custom applications                   |
| **Kinesis Data Firehose** | Delivery to S3, Elasticsearch, Splunk, or other analytics services |

### 4.3. Cross-Account Subscriptions

- A central security account can subscribe to log groups in member accounts
- Requires a **destination policy** in the target account that grants the source account permission to send logs
- Architecture: Each member account creates a subscription filter → sends logs to a central account's Kinesis or Lambda

**Exam scenario**: A security team needs to analyze logs from 100 AWS accounts in real time in a central account → use **cross-account subscription filters** sending to a Kinesis Data Streams in the central security account.

### 4.4. Subscription Filter vs Metric Filter

| Feature      | Metric Filter             | Subscription Filter                |
| ------------ | ------------------------- | ---------------------------------- |
| What it does | Creates a metric (number) | Streams raw log events             |
| Real-time    | Near-real-time (minutes)  | Real-time (seconds)                |
| Destinations | CloudWatch metric         | Lambda, Kinesis, Firehose          |
| Use case     | Alerting on thresholds    | Log aggregation, search, analytics |

## 5. CloudWatch Logs Insights

### 5.1. Purpose

- SQL-like query language for searching and analyzing log data in CloudWatch Logs
- No need to export logs to another service
- Queries run on log data stored in CloudWatch Logs (up to 15GB per query)

### 5.2. Query Syntax

```sql
fields @timestamp, @message
| filter @message like /AccessDenied/
| sort @timestamp desc
| limit 20
```

### 5.3. Common Security Queries

- Find all `AccessDenied` events in the last hour
- Identify the top 10 IP addresses generating errors
- Find all API calls made by a specific IAM user
- Correlate CloudTrail events with application logs

**Exam scenario**: A security analyst needs to search CloudTrail logs for all `DeleteBucket` events across all accounts without exporting to Athena → use **CloudWatch Logs Insights** if the logs are already in CloudWatch Logs.

## 6. CloudWatch Agent vs Unified CloudWatch Agent

| Feature            | CloudWatch Agent (legacy) | Unified CloudWatch Agent               |
| ------------------ | ------------------------- | -------------------------------------- |
| Metrics collection | Basic system metrics      | Detailed system + custom metrics       |
| Log collection     | No                        | Yes (collects logs from EC2 / on-prem) |
| OS support         | Linux + Windows           | Linux + Windows                        |
| Configuration      | Command line              | JSON configuration file                |
| Status             | Legacy                    | **Recommended**                        |

### 6.1. Unified CloudWatch Agent

- Collects system metrics (CPU, memory, disk, network) beyond the basic EC2 metrics
- Collects logs from EC2 instances and on-premises servers
- Can send logs to CloudWatch Logs in real time
- Supports **SSM (Systems Manager)** for agent configuration and management
- **Exam scenario**: An EC2 instance's application logs need to be sent to CloudWatch Logs for monitoring → install the **Unified CloudWatch Agent** on the instance.

### 6.2. IAM Permissions for the Agent

- The EC2 instance role must have permissions:
  - `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`
  - `cloudwatch:PutMetricData` (if collecting metrics)
  - `ec2:DescribeTags` (for dimension support)
- Without the correct IAM role, the agent will fail silently

## 7. Contributor Insights

### 7.1. Purpose

- Analyzes CloudWatch Logs data to identify **top contributors** — the most active sources of log events
- Answers questions like: "Which IP addresses are generating the most errors?", "Which user is making the most API calls?"

### 7.2. How It Works

- Creates **rules** that define which log fields to analyze
- Continuously analyzes log events and builds contributor rankings
- Results are visible in the Contributor Insights dashboard
- Can create alarms based on contributor metrics

**Exam scenario**: A security team needs to identify which IP address is making the highest volume of failed API calls → use **Contributor Insights** on CloudTrail logs.

## 8. Live Tail

- Real-time tailing of log events — similar to `tail -f` on Linux
- Allows filtering by pattern to focus on specific events
- Useful during incident response to watch logs in real time

## 9. Container Insights & Lambda Insights

| Feature                | Purpose                                                                       |
| ---------------------- | ----------------------------------------------------------------------------- |
| **Container Insights** | Collects metrics and logs from ECS, EKS, and Kubernetes                       |
| **Lambda Insights**    | Collects metrics and logs from Lambda functions (cold starts, duration, etc.) |

- Both are relevant for monitoring security-relevant metrics in container and serverless workloads

## 10. CloudWatch Logs Integration with Other Services

### 10.1. CloudTrail + CloudWatch Logs

- CloudTrail can deliver events to **CloudWatch Logs** in addition to S3
- This enables **real-time monitoring** of API activity via metric filters and alarms
- A service-linked role (`CloudTrail_CloudWatchLogs_Role`) is created automatically
- **Exam tip**: For real-time alerting on API calls (e.g., `StopLogging`, `CreateUser`), CloudTrail must be configured to deliver to CloudWatch Logs

### 10.2. VPC Flow Logs + CloudWatch Logs

- VPC Flow Logs can be published to CloudWatch Logs (or S3)
- Metric filters on VPC Flow Logs can detect:
  - Traffic to/from known-bad IPs
  - Rejected connections (potential scanning)
  - Unusual port usage
- **Exam tip**: VPC Flow Logs in CloudWatch Logs enables real-time network monitoring

### 10.3. AWS Config + CloudWatch Logs

- AWS Config does not directly deliver to CloudWatch Logs
- Config events can be streamed via EventBridge to Lambda, which writes to CloudWatch Logs
- Alternatively, use Config rules with remediation actions

### 10.4. Route 53 Resolver Query Logs

- DNS query logs can be published to CloudWatch Logs
- Useful for detecting DNS-based exfiltration or C2 communication
- Metric filters can detect queries to known-bad domains

## 11. CloudWatch vs Other Log Destinations

| Destination                     | Real-Time?               | Query Capability         | Long-Term Storage    | Cost         |
| ------------------------------- | ------------------------ | ------------------------ | -------------------- | ------------ |
| **CloudWatch Logs**             | Yes                      | CloudWatch Logs Insights | Yes (up to 10 years) | Medium       |
| **S3**                          | No (5-15 min delay)      | Athena, external tools   | Yes (indefinite)     | Low          |
| **CloudWatch Logs + S3 export** | Yes (CW) + archival (S3) | Insights + Athena        | Yes (indefinite)     | Medium + Low |

- Best practice: Use CloudWatch Logs for real-time monitoring and alerting, export to S3 for long-term, low-cost storage

## 12. Security Best Practices

### 12.1. Log Access Control

- Use IAM policies to restrict which users can view logs (`logs:DescribeLogGroups`, `logs:GetLogEvents`)
- Use IAM to restrict who can create/delete metric filters and subscription filters
- Use KMS encryption to separate log access from log decryption

### 12.2. Log Integrity

- Consider exporting logs to S3 with **Object Lock** (WORM) for tamper-proof long-term storage
- CloudWatch Logs itself provides append-only semantics — logs cannot be modified, only deleted (by retention policy)
- Use CloudTrail to monitor CloudWatch Logs API calls (`CreateLogGroup`, `DeleteLogGroup`, `PutMetricFilter`, `DeleteMetricFilter`)

### 12.3. Alerting

- Set up metric filters and alarms for critical security events (see section 2.2)
- Use **composite alarms** to reduce noise — combine multiple conditions
- Ensure alert destinations (SNS topics) are operational and monitored themselves

### 12.4. Cross-Account Logging

- Use **cross-account subscription filters** to aggregate logs into a central security account
- Use a **centralized logging account** pattern with organization-level logging
- Restrict which accounts can send logs to the central account via IAM and resource policies

## 13. Limits and Quotas

| Resource                                   | Limit                 |
| ------------------------------------------ | --------------------- |
| Log groups per account                     | 1,000,000             |
| Metric filters per log group               | 100                   |
| Subscription filters per log group         | 2                     |
| Log event size                             | 256 KB (max)          |
| Log retention period                       | 1 day to 10 years     |
| CloudWatch Logs Insights query concurrency | 10 concurrent queries |
| CloudWatch Logs Insights result limit      | 10,000 rows per query |
| Data volume per Insights query             | 15 GB                 |

## 14. Exam Tips

1. **CloudWatch Logs + CloudTrail** is the standard pattern for real-time security alerting. CloudTrail delivers to S3 for long-term storage, and optionally to CloudWatch Logs for real-time monitoring.

2. **Metric filters** detect patterns in logs and create metrics. They are the basis for **alarms** — the most common alerting mechanism on the exam.

3. **Subscription filters** stream raw log events to Lambda, Kinesis, or Firehose for real-time processing and aggregation.

4. **Never expire** is the default log retention — always set a retention policy for cost control and compliance.

5. **Unified CloudWatch Agent** is the recommended way to collect system metrics and application logs from EC2 instances and on-prem servers.

6. **Cross-account subscription filters** enable centralized log aggregation across multiple AWS accounts.

7. **KMS encryption** on log groups separates log access from log decryption — defense in depth.

8. **Contributor Insights** identifies top contributors (IPs, users, etc.) in log data — useful for identifying the source of anomalous activity.

9. **CloudWatch Logs Insights** enables SQL-like queries directly on CloudWatch Logs without exporting data — useful for ad-hoc investigation.

10. **Alarm evaluation**: An alarm with 3 evaluation periods and 2 datapoints to alarm is more resilient to false positives than 1 period and 1 datapoint.

11. **Composite alarms** combine multiple alarms into a single alarm — reduces alert noise.

12. **Live Tail** is useful during active incident response — real-time log streaming with filtering.

13. **VPC Flow Logs + CloudWatch Logs** enables real-time network traffic monitoring and alerting.

14. **Route 53 Resolver Query Logs** in CloudWatch Logs enables DNS-level threat detection.

15. **CloudWatch Agent IAM permissions**: The instance role needs `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`. Without these, the agent cannot send logs.

16. **CloudWatch Logs is append-only** — log events cannot be deleted or modified. Retention policies control automatic deletion. For immutable storage, export to S3 with Object Lock.
