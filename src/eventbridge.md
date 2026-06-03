# Amazon EventBridge (CloudWatch Events)

<figure>
  <img src="./images/Arch_Amazon-EventBridge_64@5x.png" alt="Amazon EventBridge Icon" width=200>
  <figcaption><center>Amazon EventBridge<br><i>Image source: AWS Documentation</i></center></figcaption>
</figure>

**Overview**: Amazon EventBridge (formerly CloudWatch Events) is a serverless event bus service that enables event-driven architecture across AWS services, custom applications, and SaaS partners. On the Security Specialty exam, EventBridge is the **central nervous system for security automation** — it routes security findings from GuardDuty, Security Hub, Config, Macie, and other services to Lambda functions, SQS queues, SNS topics, and Step Functions for automated remediation.

**Domain weight**: Appears across Detection & Logging (~18%), Incident Response (~19%), and Infrastructure Security (~17%). EventBridge is not a security service itself but is the **glue that connects security detection to automated response**.

**Naming note**: CloudWatch Events is the predecessor. EventBridge is the modern evolution with additional capabilities (cross-account event buses, schema registry, Pipes). The exam may use both names; EventBridge is the current AWS-recommended service.

## 1. Core Concepts

### 1.1. Event Bus

- An **event bus** receives events and routes them to targets based on rules
- Three types of event buses:

| Bus Type    | Description                                                            | Use Case                                     |
| ----------- | ---------------------------------------------------------------------- | -------------------------------------------- |
| **Default** | Automatically created per account; receives events from AWS services   | AWS service events (GuardDuty, Config, etc.) |
| **Custom**  | Created by you; receives events from your applications                 | Custom application events                    |
| **Partner** | Receives events from SaaS partners (Datadog, Zendesk, PagerDuty, etc.) | Third-party integrations                     |

### 1.2. Events

- An **event** is a JSON object representing a change in state
- Structure: `source`, `detail-type`, `detail`, `resources`, `time`, `region`, `account`, `id`
- Example GuardDuty finding event:

```json
{
  "version": "0",
  "id": "12345678-1234-1234-1234-123456789012",
  "detail-type": "GuardDuty Finding",
  "source": "aws.guardduty",
  "account": "123456789012",
  "time": "2024-01-15T10:00:00Z",
  "region": "us-east-1",
  "resources": [
    "arn:aws:ec2:us-east-1:123456789012:instance/i-1234567890abcdef0"
  ],
  "detail": {
    "severity": 8.0,
    "type": "Recon:EC2/Portscan",
    "id": "arn:aws:guardduty:us-east-1:123456789012:detector/...",
    "title": "EC2 instance has a port scan activity.",
    "description": "..."
  }
}
```

### 1.3. Rules

- Rules **match incoming events** and route them to **targets**
- A rule consists of:
  - **Event pattern**: JSON pattern to match against events
  - **Targets**: Where to send matching events (Lambda, SQS, SNS, Step Functions, etc.)
  - **State**: Enabled or disabled
- Rules are associated with a single event bus
- Multiple rules can match the same event (fan-out)

### 1.4. Targets

| Target Type                      | Description                                         |
| -------------------------------- | --------------------------------------------------- |
| **Lambda Function**              | Invoke a Lambda function asynchronously             |
| **SQS Queue**                    | Send event to an SQS queue (for durable processing) |
| **SNS Topic**                    | Fan out to multiple subscribers                     |
| **Step Functions**               | Start a state machine execution                     |
| **Kinesis Stream**               | Send event to a Kinesis stream                      |
| **Systems Manager**              | Run Automation or Run Command                       |
| **ECS Task**                     | Run an ECS task                                     |
| **EventBridge API Destinations** | Call external HTTP endpoints (webhooks)             |
| **EC2 Action**                   | Stop, terminate, reboot, or recover an EC2 instance |

**Exam scenario**: EventBridge is the **most common trigger for security automation Lambda functions** — GuardDuty finding → EventBridge → Lambda (isolate instance, revoke keys, etc.).

## 2. Event Patterns

### 2.1. Pattern Matching

- Event patterns are JSON objects that match against incoming events
- Patterns can match on: `source`, `detail-type`, `detail` fields, `resources`, and more
- If an event matches the pattern, it is routed to all targets of the rule

**Simple match**: Match all GuardDuty findings:

```json
{
  "source": ["aws.guardduty"],
  "detail-type": ["GuardDuty Finding"]
}
```

**Specific match**: Match only high-severity GuardDuty findings:

```json
{
  "source": ["aws.guardduty"],
  "detail-type": ["GuardDuty Finding"],
  "detail": {
    "severity": [{ "numeric": [">=", 7.0] }]
  }
}
```

### 2.2. Pattern Matching Features

| Feature                | Example                                         | Description                          |
| ---------------------- | ----------------------------------------------- | ------------------------------------ |
| **Exact match**        | `"source": ["aws.guardduty"]`                   | Match exact value                    |
| **Prefix match**       | `"source": [{"prefix": "aws."}]`                | Match any value starting with prefix |
| **Numeric match**      | `"severity": [{"numeric": [">=", 7.0]}]`        | Numeric comparison (>, >=, <, <=, =) |
| **Exists match**       | `"detail": {"ipAddress": [{"exists": true}]}`   | Check if a field exists              |
| **Anything-but match** | `"source": [{"anything-but": "aws.guardduty"}]` | Match anything except this value     |
| **Or logic**           | `"source": ["aws.guardduty", "aws.macie"]`      | Match any value in the list          |
| **And logic**          | Deeply nested JSON fields                       | All specified fields must match      |

**Exam scenario**: A security team needs to trigger a Lambda function only for GuardDuty findings with severity ≥ 7.0 (HIGH) → use **EventBridge rule** with an event pattern matching `source: aws.guardduty` AND `detail.severity >= 7.0`.

## 3. Security Automation with EventBridge (Exam Critical)

### 3.1. Common Security Event Sources

| AWS Service         | `source` Value       | `detail-type` Examples                                                         |
| ------------------- | -------------------- | ------------------------------------------------------------------------------ |
| **GuardDuty**       | `aws.guardduty`      | `GuardDuty Finding`                                                            |
| **Security Hub**    | `aws.securityhub`    | `Security Hub Findings - Imported`, `Security Hub Insight Results`             |
| **AWS Config**      | `aws.config`         | `Config Rules ComplianceChange`, `Config ConfigurationSnapshotDeliveryStarted` |
| **Macie**           | `aws.macie`          | `Macie Finding`                                                                |
| **IAM**             | `aws.iam`            | `IAM Policy Change` (via CloudTrail)                                           |
| **CloudTrail**      | `aws.cloudtrail`     | `AWS API Call via CloudTrail`                                                  |
| **S3**              | `aws.s3`             | `Object Created`, `Object Removed`, etc.                                       |
| **Inspector**       | `aws.inspector`      | `Inspitor Finding`                                                             |
| **WAF**             | `aws.waf`            | `WAF WebACL` changes                                                           |
| **Health**          | `aws.health`         | `AWS Health Event`                                                             |
| **Trusted Advisor** | `aws.trustedadvisor` | `Trusted Advisor Check Item Refresh`                                           |

### 3.2. Security Automation Patterns

| Pattern                             | Event Source            | Target         | Action                                                   |
| ----------------------------------- | ----------------------- | -------------- | -------------------------------------------------------- |
| Isolate compromised EC2             | GuardDuty finding       | Lambda         | Stop/isolate the EC2 instance, update security groups    |
| Revoke compromised IAM keys         | GuardDuty finding       | Lambda         | Disable access key, attach deny-all policy               |
| Remediate non-compliant resource    | Config compliance       | Lambda         | Apply remediation (e.g., enable encryption, close ports) |
| Alert on critical findings          | Security Hub            | SNS            | Send notification to security team via email/SMS         |
| Automate incident response playbook | GuardDuty finding       | Step Functions | Execute multi-step response workflow                     |
| Block malicious IPs                 | GuardDuty/WAF finding   | Lambda         | Update WAF IP set or Network ACL                         |
| Rotate exposed credentials          | GuardDuty finding       | Lambda         | Call Secrets Manager to rotate the secret                |
| Disable unused access keys          | IAM change (CloudTrail) | Lambda         | Detect and disable keys not used in 90 days              |

**Exam scenario**: An organization needs to automatically isolate EC2 instances that are flagged by GuardDuty for crypto mining → GuardDuty → **EventBridge** (filter for `UnauthorizedAccess:EC2/CryptoCurrency` with severity ≥ 7) → Lambda (isolate the instance).

**Exam scenario**: A security team wants to automatically respond to Security Hub findings of type `S3.1` (S3 bucket public read) → Security Hub → **EventBridge** → Lambda (apply Block Public Access).

## 4. Cross-Account Event Buses

### 4.1. Centralized Security Event Collection

- EventBridge supports **cross-account event buses** — events from one account can be sent to an event bus in another account
- Enables **centralized security monitoring**: all GuardDuty/Security Hub/Config findings from all accounts sent to a central security account
- The target account must have a **resource-based policy** on the event bus
- The source account's rule must have the central event bus as a target

### 4.2. Architecture

```
Account A (Member)  ──Event──→ Account B (Security Central)
  GuardDuty finding  ──Rule──→  Central Event Bus → Lambda/SNS/Step Functions
```

### 4.3. Resource-Based Policy on Event Bus

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowMemberAccounts",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:root"
      },
      "Action": "events:PutEvents",
      "Resource": "arn:aws:events:us-east-1:222222222222:event-bus/default"
    }
  ]
}
```

- The central security account (222222222222) adds a resource-based policy to its event bus
- Member account (111111111111) creates a rule that targets the central event bus
- The member account's rule uses the central event bus ARN as the target

**Exam scenario**: An organization with 50 AWS accounts needs to send all GuardDuty findings to a central security account for analysis → use a **cross-account EventBridge rule** in each account targeting the central security account's event bus.

### 4.4. Organization Event Buses

- EventBridge can automatically replicate events across all accounts in an **AWS Organization**
- No need to configure individual cross-account rules per account
- Requires EventBridge to be enabled as a **trusted service** in Organizations
- Much simpler than per-account cross-account event bus configuration

**Exam scenario**: A large organization with hundreds of AWS accounts needs to centralize security events without manually configuring each account → use **EventBridge Organization event bus** (or AWS Organizations + delegated admin for Security Hub/GuardDuty which natively aggregate).

## 5. EventBridge Scheduler

### 5.1. Overview

- **EventBridge Scheduler** (launched 2022) replaces CloudWatch Events scheduled rules
- Used for **time-based** event scheduling (cron or rate expressions)
- Can invoke: Lambda, SQS, SNS, Step Functions, EventBridge API Destinations, and more
- More flexible than the old scheduled rules — supports timezones, flexible time windows, and retry policies

### 5.2. Security Use Cases

| Use Case                            | Schedule              | Target                 |
| ----------------------------------- | --------------------- | ---------------------- |
| Daily scan of S3 for public buckets | `cron(0 6 * * ? *)`   | Lambda                 |
| Weekly IAM access key audit         | `cron(0 8 ? * MON *)` | Lambda                 |
| Hourly log rotation / archival      | `rate(1 hour)`        | Lambda/Step Functions  |
| Periodic compliance check           | `rate(6 hours)`       | Lambda (invoke Config) |

**Exam scenario**: A security team needs to run an S3 bucket audit every Monday at 8 AM → use **EventBridge Scheduler** with a `cron(0 8 ? * MON *)` schedule targeting a Lambda function.

## 6. EventBridge Pipes

### 6.1. Overview

- **EventBridge Pipes** provides a point-to-point integration between event sources and targets
- Unlike rules (which use a bus), Pipes connect a **single source** to a **single target**
- Supports sources: SQS, DynamoDB Streams, Kinesis, MSK, Self-managed Kafka
- Supports optional filtering, enrichment (with Lambda, Step Functions, or API Destinations), and transformation
- Useful for building **security data pipelines** (e.g., CloudTrail logs → enrichment → SIEM)

### 6.2. Pipes vs Events

| Feature        | EventBridge Events (Rules)       | EventBridge Pipes                           |
| -------------- | -------------------------------- | ------------------------------------------- |
| **Source**     | Event bus (multiple sources)     | Single source (SQS, DynamoDB, Kinesis)      |
| **Target**     | Multiple targets (fan-out)       | Single target                               |
| **Enrichment** | ❌ Not supported                 | ✅ Lambda, Step Functions, API Destinations |
| **Filtering**  | ✅ Event patterns                | ✅ Event patterns                           |
| **Use case**   | Event-driven security automation | Data pipeline for security events           |

## 7. EventBridge vs Other Services

### 7.1. EventBridge vs SQS

| Feature            | EventBridge                            | SQS                                        |
| ------------------ | -------------------------------------- | ------------------------------------------ |
| **Purpose**        | Route events to multiple targets       | Decouple applications with a message queue |
| **Delivery model** | Push (to targets)                      | Pull (consumers poll the queue)            |
| **Persistence**    | Events not stored (routed immediately) | Messages persisted until deleted           |
| **Retry**          | Limited (configurable retry policy)    | Durable (visibility timeout, DLQ)          |
| **Fan-out**        | ✅ Yes (multiple targets per rule)     | ❌ No (one consumer per message)           |
| **Scheduling**     | ✅ Yes (EventBridge Scheduler)         | ❌ No                                      |
| **Cross-account**  | ✅ Yes (cross-account event buses)     | ✅ Yes (SQS policy)                        |

### 7.2. EventBridge vs SNS

| Feature           | EventBridge                              | SNS                                         |
| ----------------- | ---------------------------------------- | ------------------------------------------- |
| **Purpose**       | Event routing with filtering and targets | Pub/sub messaging with fan-out              |
| **Filtering**     | ✅ Rich JSON pattern matching            | ✅ Basic message filtering per subscription |
| **Targets**       | Many AWS services + API Destinations     | Lambda, SQS, HTTP, Email, SMS, Mobile Push  |
| **Cross-account** | ✅ Cross-account event buses             | ✅ SNS topic policy                         |
| **Schema**        | ✅ Schema Registry                       | ❌ Not supported                            |
| **Use case**      | Event-driven automation                  | Notification delivery                       |

**Exam tip**: On the exam, EventBridge is typically the answer when the scenario involves **event-driven security automation** — a finding needs to trigger an action. SNS is for notifications. SQS is for decoupled processing.

## 8. EventBridge API Destinations

### 8.1. Overview

- Send events to **external HTTP endpoints** (third-party APIs, webhooks)
- Supports authentication: API key, OAuth2, Basic Auth
- Uses **connections** to securely store credentials
- Connection credentials are encrypted with KMS
- Rate limiting per destination

### 8.2. Security Use Case

- Send security alerts to external SIEM (Splunk, Sumo Logic)
- Create a PagerDuty incident from a GuardDuty finding
- Post to a Slack webhook when a security event occurs
- Trigger a third-party SOAR platform

**Exam scenario**: A security team wants to automatically create a PagerDuty incident when GuardDuty detects a critical finding → use **EventBridge API Destination** with an API key connection to PagerDuty.

## 9. Archiving and Replaying Events

### 9.1. Archive

- EventBridge can **archive** events to an S3 bucket
- Archive configuration specifies: event pattern filter, retention period, and destination
- Archives can be replayed to generate events again

### 9.2. Replay

- Events in an archive can be **replayed** to the same event bus
- Useful for:
  - Reprocessing security events after a bug fix
  - Testing security automation after changes
  - Auditing past security incidents

**Exam scenario**: A security automation Lambda function had a bug and failed to process 24 hours of GuardDuty findings → replay the events from the **EventBridge archive** after the bug is fixed.

## 10. Monitoring and Auditing

### 10.1. CloudTrail

- All EventBridge API calls are logged in CloudTrail:
  - `PutRule` (create/update rule)
  - `PutTargets` (add targets to rule)
  - `PutEvents` (ingest events)
  - `CreateEventBus` (create custom bus)
  - `CreateArchive` (create archive)
- Monitor for changes to security automation rules

### 10.2. CloudWatch Metrics

| Metric              | Description                             |
| ------------------- | --------------------------------------- |
| `Invocations`       | Number of times a rule invokes a target |
| `FailedInvocations` | Number of failed invocations            |
| `ThrottledRules`    | Number of throttled rule evaluations    |
| `TriggeredRules`    | Number of rules that matched events     |
| `MatchedEvents`     | Number of events that matched a rule    |

### 10.3. Dead Letter Queue (DLQ)

- Rules can send failed events to a **DLQ** (SQS queue)
- If a target is unavailable or returns an error, the event goes to the DLQ
- Configured per rule (as a target parameter)
- Use SQS DLQ for durable, reprocessable event storage

**Exam scenario**: A critical security automation rule occasionally fails to invoke the target Lambda function. Events are being lost → configure a **dead letter queue** on the rule to capture failed events.

## 11. Security Best Practices

| Practice                                                     | Description                                                            |
| ------------------------------------------------------------ | ---------------------------------------------------------------------- |
| **Use least-privilege on event bus policies**                | Restrict which accounts/services can put events                        |
| **Use specific event patterns**                              | Avoid broad patterns that could trigger unintended actions             |
| **Enable DLQ on critical rules**                             | Capture failed invocations for reprocessing                            |
| **Monitor Invocations and FailedInvocations metrics**        | Detect when security automation is failing                             |
| **Use cross-account event buses for centralized monitoring** | Aggregate security events into a single security account               |
| **Use API Destinations for external integrations**           | Securely send events to third-party SIEM/SOAR with KMS encryption      |
| **Archive security events**                                  | Retain event history for forensic analysis and replay                  |
| **Use CloudTrail to monitor rule changes**                   | Detect unauthorized changes to security automation rules               |
| **Apply tags to rules and event buses**                      | Organize and control access by environment or team                     |
| **Use input transformers**                                   | Protect sensitive data in events by transforming the payload           |
| **Set appropriate retry policies**                           | Avoid infinite retries for unrecoverable errors                        |
| **Integrate with Security Hub**                              | Route Security Hub findings through EventBridge for automated response |

## 12. Common Exam Scenarios

1. **GuardDuty → EventBridge → Lambda**: The most common security automation pattern. GuardDuty detects a threat, EventBridge routes the finding, Lambda performs remediation.

2. **Security Hub → EventBridge → SNS**: Centralized alerting — Security Hub aggregates findings from GuardDuty, Inspector, Macie, and Config, EventBridge routes critical findings to SNS for email/SMS notification.

3. **Config → EventBridge → Lambda**: Auto-remediation — when a resource becomes non-compliant (e.g., S3 bucket becomes public), Config sends a compliance change event, EventBridge triggers Lambda to remediate.

4. **Cross-account event aggregation**: All accounts send security events to a central security account using cross-account event buses. This is tested directly.

5. **EventBridge Scheduler for periodic scans**: Schedule recurring security audits (e.g., "run IAM access key audit every Monday at 8 AM").

6. **EventBridge archive + replay**: Reprocess security events after fixing a bug in the automation function.

7. **API Destination for PagerDuty**: Send critical security events to an external incident management platform.

8. **Scheduled rule for log cleanup**: Run a Lambda function daily to clean up old CloudWatch Log groups or S3 logs.

9. **DLQ for failed automation**: Ensure no security events are lost by adding a DLQ to critical rules.

10. **EventBridge Pipes for security data pipelines**: Stream CloudTrail logs to a Lambda enrichment function, then to a SIEM.

## 13. Exam Tips

1. **EventBridge is the glue**: It connects security detections (GuardDuty, Security Hub, Config) to automated responses (Lambda, Step Functions, SNS).

2. **Event patterns**: The exam may ask you to write or interpret an event pattern. Remember: `source` + `detail-type` + `detail` fields.

3. **Cross-account event buses**: Centralize security events from member accounts → use resource-based policy on the central event bus.

4. **Organization event bus**: Even simpler than cross-account — works with AWS Organizations for automatic event replication.

5. **EventBridge Scheduler**: For scheduled/cron-based security tasks (replaces CloudWatch Events scheduled rules).

6. **EventBridge Pipes**: For point-to-point streaming pipelines. Less common on the exam than Rules.

7. **API Destinations**: Securely call external APIs (SIEM, PagerDuty, Slack) — credentials encrypted with KMS.

8. **DLQ**: Use a dead letter queue on critical rules to capture failed invocations. Never lose a security event.

9. **Archive + Replay**: Events can be archived and replayed — useful for reprocessing after fixing automation bugs.

10. **Input transformers**: Modify event payloads before sending to targets — can protect sensitive data.

11. **CloudTrail monitoring of EventBridge**: Monitor `PutRule`, `PutTargets`, `PutEvents` for unauthorized changes.

12. **InvocationFailed metric**: Create CloudWatch alarms on `FailedInvocations` for critical rules.

13. **Multiple targets per rule**: One rule can fan-out to Lambda + SQS + SNS simultaneously.

14. **EventBridge is regional**: Events stay within the region. Use cross-account event buses to send events between regions.

15. **Retry policy**: Configurable — event is retried for up to 24 hours (default) with exponential backoff.

16. **Event size limit**: 256 KB per event (same as CloudWatch Events). Larger events are rejected.

17. **Events schema**: EventBridge has a Schema Registry to discover and generate code for event schemas.

18. **SaaS partner events**: EventBridge integrates with Datadog, PagerDuty, Zendesk, New Relic, and 90+ other SaaS partners.

19. **CloudWatch Events compatibility**: EventBridge is backward-compatible with CloudWatch Events APIs and formats.

20. **EventBridge in security accounts**: The recommended pattern is to have a central security account with a default event bus that receives security events from all member accounts.
