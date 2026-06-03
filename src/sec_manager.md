# AWS Secrets Manager

<figure>
  <img src="./images/Arch_AWS-Secrets-Manager_64@5x.png" alt="AWS Secrets Manager Icon" width=200>
  <figcaption><center>AWS Secrets Manager<br><i>Image source: AWS Documentation</i></center></figcaption>
</figure>

**Overview**: AWS Secrets Manager is a fully managed service for securely storing, managing, and **automatically rotating** secrets such as database credentials, API keys, OAuth tokens, and other sensitive information. Secrets are encrypted at rest with KMS and accessed via IAM permissions. The key differentiator from SSM Parameter Store is **automatic secret rotation** — Secrets Manager can rotate secrets on a schedule without application downtime.

**Domain weight**: Appears across Data Protection (~18%) and IAM (~20%) domains. The exam tests Secrets Manager primarily via comparison questions with SSM Parameter Store and rotation scenario questions.

## 1. Secret Structure

### 1.1. Secret Components

| Component           | Description                                                                            |
| ------------------- | -------------------------------------------------------------------------------------- |
| **Secret name**     | Friendly name for the secret (e.g., `prod/db/password`)                                |
| **Secret value**    | The encrypted secret data (JSON key-value pairs or plaintext)                          |
| **ARN**             | Unique identifier: `arn:aws:secretsmanager:<region>:<account>:secret:<name>-<random6>` |
| **KMS key**         | Used to encrypt/decrypt the secret at rest                                             |
| **Rotation config** | Lambda function ARN + rotation interval + schedule expression                          |
| **Tags**            | Metadata labels for the secret                                                         |
| **Version IDs**     | Staging labels track rotation stages (AWSCURRENT, AWSPREVIOUS, AWSPENDING)             |

### 1.2. Secret Value Formats

Secrets can store:

- **Plaintext** (string)
- **JSON** (key-value pairs) — most common for database credentials:
  ```json
  {
    "username": "admin",
    "password": "p@ssw0rd",
    "engine": "mysql",
    "host": "mydb.example.com",
    "port": 3306,
    "dbInstanceIdentifier": "mydb"
  }
  ```

### 1.3. Versioning and Staging Labels

- Secrets Manager uses **staging labels** to track versions (not sequential version IDs)
- Three built-in staging labels:

| Label         | Purpose                                                         |
| ------------- | --------------------------------------------------------------- |
| `AWSCURRENT`  | The current/active version of the secret                        |
| `AWSPREVIOUS` | The previous version (before the last rotation)                 |
| `AWSPENDING`  | The future version (during rotation, before it becomes current) |

- Staging labels can be moved between versions manually
- Each secret version has a unique **Version ID** (a UUID)

**Exam scenario**: During rotation, `AWSPENDING` is created first, tested, then promoted to `AWSCURRENT`. The old `AWSCURRENT` becomes `AWSPREVIOUS`. If rotation fails, the old `AWSCURRENT` is preserved.

## 2. Encryption

### 2.1. Encryption at Rest (SSE-KMS)

- All secrets are encrypted at rest using **AWS KMS**
- By default, Secrets Manager uses the **AWS managed key** `aws/secretsmanager`
- You can specify a **customer managed KMS key** for tighter control:
  - Separate permissions for who can decrypt secrets
  - Key rotation control
  - CloudTrail audit of KMS `Decrypt` calls
  - Ability to use `kms:ViaService` condition to restrict key usage

### 2.2. Encryption in Transit

- All API calls to Secrets Manager use **TLS/HTTPS** (encryption in transit)
- Secrets are transmitted **only over encrypted channels**
- When retrieved, the secret is decrypted server-side and returned over TLS

**Exam scenario**: A compliance requirement mandates encryption keys for secrets to be managed by the customer → use a **customer managed KMS key** in Secrets Manager instead of the default `aws/secretsmanager` key.

## 3. Automatic Secret Rotation (Key Differentiator)

### 3.1. Overview

- Secrets Manager can **automatically rotate** secrets on a configurable schedule
- This is the **primary differentiator** from SSM Parameter Store (which does not support rotation)
- Supported for: **Amazon RDS** (MySQL, PostgreSQL, Oracle, SQL Server, MariaDB), **Amazon Redshift**, **Amazon DocumentDB**, and **custom** (any service via a Lambda rotation function)
- Rotation uses a **Lambda function** that Secrets Manager invokes
- Rotation does **not require application changes** — apps read `AWSCURRENT` and get the rotated secret automatically

### 3.2. Rotation Schedule

| Setting                 | Default         | Notes                                              |
| ----------------------- | --------------- | -------------------------------------------------- |
| **Rotation interval**   | 30 days         | Can be set to any number of days (minimum? varies) |
| **Schedule expression** | `rate(30 days)` | Can also use `cron()` expressions                  |
| **Immediate rotation**  | Not automatic   | Must rotate immediately via API or console         |

- Rotation can be **turned on/off** per secret
- Rotation can be **triggered immediately** using `RotateSecret` API
- If you change the rotation interval, the next rotation is calculated from the last rotation date

### 3.3. Rotation Process (Standard Two-Phase)

Secrets Manager supports multiple rotation strategies. The standard approach:

**Phase 1: Create Pending Version**

1. Secrets Manager calls `lambda.invoke()` on the rotation Lambda function
2. Lambda generates a new secret (new password for the database user, new API key, etc.)
3. Lambda stores the new secret as `AWSPENDING` (encrypted, not yet current)
4. Lambda updates the database/service with the new credentials
5. Lambda tests the new credentials

**Phase 2: Make Pending Current** 6. Secrets Manager promotes `AWSPENDING` → `AWSCURRENT` 7. The old `AWSCURRENT` becomes `AWSPREVIOUS` 8. Lambda can optionally test the old credentials still work (graceful transition)

### 3.4. Rotation Strategies

| Strategy                          | Description                                                                   | Use Case                                                                       |
| --------------------------------- | ----------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| **Single user**                   | Rotates the password of a single database user                                | Simple applications; brief downtime risk if credential is used during rotation |
| **Alternating users** (dual-user) | Rotates passwords for two users alternately; one always has valid credentials | Zero-downtime rotation; app uses one user while the other is being rotated     |

- **Alternating users strategy** is preferred for production — it ensures credentials are always valid
- The Lambda function creates a clone of the database user with the new password, then switches

**Exam scenario**: A production database cannot have even a second of downtime during credential rotation → use **alternating users (dual-user) rotation strategy**.

### 3.5. Rotation Lambda Function

- Each rotation-enabled secret has a custom Lambda function
- AWS provides **managed Lambda templates** for RDS, Redshift, and DocumentDB
- The Lambda function needs permissions to:
  - `secretsmanager:GetSecretValue`
  - `secretsmanager:PutSecretValue`
  - `secretsmanager:UpdateSecretVersionStage`
  - `kms:Decrypt` (to decrypt the secret)
  - Database access (to change the password)
- For custom secrets, you write your own Lambda function

**Exam scenario**: A team wants to rotate API keys stored in Secrets Manager for a third-party service → create a **custom Lambda rotation function** that calls the third-party API to generate a new key and update the secret.

### 3.6. Rotation Events (EventBridge)

- Secrets Manager emits events to **Amazon EventBridge** for rotation lifecycle:
  - `RotationStarted`, `RotationSucceeded`, `RotationFailed`
  - `SecretVersionDeletion`, `SecretVersionStageChanged`
- Can trigger notifications, remediation workflows, or monitoring alerts

## 4. Access Control

### 4.1. IAM Permissions

| Action                          | Description                                 |
| ------------------------------- | ------------------------------------------- |
| `secretsmanager:CreateSecret`   | Create a new secret                         |
| `secretsmanager:GetSecretValue` | Retrieve the decrypted secret value         |
| `secretsmanager:PutSecretValue` | Update a secret value                       |
| `secretsmanager:RotateSecret`   | Trigger manual rotation                     |
| `secretsmanager:DeleteSecret`   | Delete a secret (scheduled deletion)        |
| `secretsmanager:ListSecrets`    | List all secrets in the account             |
| `secretsmanager:DescribeSecret` | Get metadata about a secret (not the value) |
| `secretsmanager:TagResource`    | Add/update tags on a secret                 |

### 4.2. Resource-Based Policies

- Secrets Manager supports **resource-based policies** attached directly to a secret
- Allows **cross-account access** without creating an IAM role
- Maximum policy size: 20 KB (same as IAM)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "*",
      "Condition": {
        "ForAnyValue:StringEquals": {
          "secretsmanager:VersionStage": "AWSCURRENT"
        }
      }
    }
  ]
}
```

**Exam scenario**: Account B needs to read a secret stored in Account A → Account A attaches a **resource-based policy** to the secret allowing Account B, AND Account B creates an IAM policy allowing `secretsmanager:GetSecretValue`.

### 4.3. Condition Keys

| Condition Key                      | Description                                                                              |
| ---------------------------------- | ---------------------------------------------------------------------------------------- |
| `secretsmanager:VersionStage`      | Restrict access to secrets with a specific staging label (`AWSCURRENT` or `AWSPREVIOUS`) |
| `secretsmanager:VersionId`         | Restrict access to a specific version of the secret                                      |
| `secretsmanager:ResourceTag/<key>` | Restrict based on secret tags                                                            |
| `aws:ResourceTag/<key>`            | Global condition for resource tags                                                       |
| `aws:SourceIp`                     | Restrict by IP address                                                                   |
| `aws:SourceVpce`                   | Restrict by VPC Endpoint ID                                                              |
| `aws:SourceVpc`                    | Restrict by VPC ID                                                                       |
| `aws:RequestTag/<key>`             | Restrict tagging actions                                                                 |
| `aws:TagKeys`                      | Restrict which tag keys can be used                                                      |

**Exam scenario**: An application should only be able to retrieve the **current** version of a secret (not previous versions) → use the `secretsmanager:VersionStage` condition key requiring `AWSCURRENT`.

## 5. Multi-Region Secrets

- Secrets can be **replicated** to multiple AWS Regions
- The primary secret is in one region; **replica secrets** are created in other regions
- When the primary secret is rotated, replicas are **automatically updated** (within minutes)
- Replica secrets have the same **KMS key or a different KMS key per region**
- Replicas are **read-only** — rotation is initiated from the primary
- Use cases:
  - Disaster recovery (secrets available in secondary region)
  - Multi-Region applications (separate databases per region)
  - Compliance (secrets must be in-region)

**Exam scenario**: A global application running in multiple AWS Regions needs the same database credentials in all regions → create a **primary secret in one region and replicate** to all other regions. Rotation happens on the primary, replicas update automatically.

## 6. Secret Deletion and Recovery

### 6.1. Deletion Process

- Deleting a secret puts it in a **pending deletion** state
- Default waiting period: **30 days** (configurable 7–30 days)
- During the waiting period, the secret cannot be used (returns `ResourceNotFoundException`)
- After the waiting period, the secret is **permanently deleted**
- A secret in pending deletion has the `DeletedDate` field set

### 6.2. Recovery

- During the waiting period, the secret can be **restored** using `RestoreSecret`
- After permanent deletion, the secret **cannot be recovered**
- To immediately delete without recovery: `ForceDeleteWithoutRecovery` (not recommended)

**Exam scenario**: A secret was accidentally deleted 10 days ago, and the application is failing → if the deletion is still within the 30-day waiting period, use **RestoreSecret** to recover it. If more than 30 days have passed, the secret is permanently gone.

## 7. Secrets Manager vs SSM Parameter Store

This is one of the most frequently tested comparison topics on the exam.

| Feature                       | Secrets Manager                             | SSM Parameter Store                         |
| ----------------------------- | ------------------------------------------- | ------------------------------------------- |
| **Purpose**                   | Secrets (passwords, keys, tokens)           | Configuration data and secrets              |
| **Max secret/parameter size** | 64 KB (10 KB for GetSecretValue JSON)       | 4 KB (Standard), 8 KB (Advanced)            |
| **Encryption**                | Always encrypted with KMS                   | Optional (SecureString with KMS)            |
| **Automatic rotation**        | ✅ Yes (built-in with Lambda)               | ❌ No                                       |
| **Rotation Lambda**           | Built-in templates for RDS, Redshift, DocDB | Not supported                               |
| **Cross-account access**      | Resource-based policies                     | Not natively (must use IAM role assumption) |
| **Versioning**                | Staging labels (AWSCURRENT, AWSPREVIOUS)    | Sequential version numbers                  |
| **Cost**                      | $0.40/secret/month + rotation costs         | Standard: free; Advanced: $0.05/param/month |
| **CloudFormation support**    | Full support                                | Full support                                |
| **Tag-based access control**  | ✅ Yes                                      | ✅ Yes                                      |
| **Key rotation**              | Manual (rotate the KMS key)                 | Manual (rotate the KMS key)                 |
| **API call cost**             | $0.05 per 10,000 API calls                  | Free (Standard), $0.05 per 10k (Advanced)   |
| **Integration with KMS**      | Always (SSE-KMS)                            | Optional (SecureString)                     |

### 7.1. When to Use Which

| Scenario                                      | Choose                                            |
| --------------------------------------------- | ------------------------------------------------- |
| Store database credentials with auto-rotation | **Secrets Manager**                               |
| Store API keys that need periodic rotation    | **Secrets Manager**                               |
| Store application configuration (non-secret)  | **SSM Parameter Store**                           |
| Store secrets that do NOT need rotation       | Either (cost may favor Parameter Store)           |
| Store secrets up to 8 KB, no rotation needed  | **SSM Parameter Store (Advanced)**                |
| Store secrets up to 64 KB                     | **Secrets Manager**                               |
| Cross-account secret access                   | **Secrets Manager**                               |
| Free option for secrets                       | **SSM Parameter Store (Standard - SecureString)** |
| Hierarchical parameter structure              | **SSM Parameter Store**                           |

**Exam scenario**: An application needs to store database passwords and rotate them every 30 days without downtime → use **AWS Secrets Manager** with the alternating users rotation strategy.

**Exam scenario**: Store application configuration data (URLs, feature flags) that is not sensitive → use **SSM Parameter Store** (free for Standard parameters).

**Exam scenario**: The team needs to automatically rotate RDS database credentials on a 30-day schedule → use **AWS Secrets Manager** with the built-in RDS rotation Lambda template.

## 8. Secrets Manager Integration with AWS Services

| Service                | Integration                                                                            |
| ---------------------- | -------------------------------------------------------------------------------------- |
| **Amazon RDS**         | Auto-rotation of database credentials (MySQL, PostgreSQL, Oracle, SQL Server, MariaDB) |
| **Amazon Redshift**    | Auto-rotation of admin user credentials                                                |
| **Amazon DocumentDB**  | Auto-rotation of cluster credentials                                                   |
| **Amazon ECS/EKS**     | Inject secrets as environment variables or volumes                                     |
| **AWS Lambda**         | Retrieve secrets within function code via SDK/API                                      |
| **AWS CloudFormation** | Create secrets with `AWS::SecretsManager::Secret` resource                             |
| **AWS CloudTrail**     | Logs all Secrets Manager API calls                                                     |
| **Amazon EventBridge** | Rotation events (start, success, failure)                                              |
| **Amazon SNS**         | Notifications on rotation failures or secret changes                                   |
| **AWS KMS**            | Encrypts all secrets at rest                                                           |

**Exam scenario**: A Lambda function needs to access a database — it should retrieve the credentials from **Secrets Manager** at runtime (not hardcoded in environment variables).

## 9. Cost Considerations

| Cost Component           | Price                                 | Notes                                |
| ------------------------ | ------------------------------------- | ------------------------------------ |
| **Per secret**           | $0.40 per secret per month            | Each secret regardless of size       |
| **API calls**            | $0.05 per 10,000 API calls            | GetSecretValue, PutSecretValue, etc. |
| **Rotation Lambda**      | AWS Lambda costs apply                | Based on execution time and memory   |
| **Multi-Region replica** | $0.40 per secret per region per month | Each replica is billed separately    |
| **KMS usage**            | KMS costs apply                       | KMS key usage + API calls            |

**Exam tip**: Secrets Manager is more expensive than SSM Parameter Store. If the requirement does not need rotation, Parameter Store may be the cost-effective answer.

## 10. Monitoring and Auditing

### 10.1. CloudTrail

- All Secrets Manager API calls are logged in CloudTrail
- Key events to monitor:
  - `GetSecretValue` (especially from unknown IPs or roles)
  - `RotateSecret` (manual or automatic)
  - `PutSecretValue` (secret updated)
  - `DeleteSecret` (scheduled or force deletion)
  - `RestoreSecret` (secret recovered)

### 10.2. CloudWatch Metrics

- Secrets Manager emits metrics to CloudWatch:
  - `RotationSucceeded` (count)
  - `RotationFailed` (count)
  - `SecretVersionStageChanged` (count)

### 10.3. EventBridge Events

| Event Type             | Detail Type                                              |
| ---------------------- | -------------------------------------------------------- |
| Rotation lifecycle     | `RotationStarted`, `RotationSucceeded`, `RotationFailed` |
| Secret version changes | `SecretVersionStageChanged`                              |
| Secret deletion        | `SecretDeleted`                                          |

### 10.4. AWS Config

- AWS Config can track Secrets Manager resources and configuration changes
- Custom Config rules can validate:
  - Secrets must have rotation enabled
  - Secrets must use customer managed KMS keys
  - Secrets must not have public resource-based policies

## 11. Security Best Practices

| Practice                                                  | Description                                                  |
| --------------------------------------------------------- | ------------------------------------------------------------ |
| **Enable automatic rotation**                             | Rotate secrets on a schedule (default: 30 days)              |
| **Use customer managed KMS key**                          | Separate permissions, rotation control, audit trail          |
| **Restrict GetSecretValue access**                        | Grant only to roles that need it; use condition keys         |
| **Use resource-based policies for cross-account**         | Instead of IAM role assumption (simpler for Secrets Manager) |
| **Monitor GetSecretValue in CloudTrail**                  | Detect unauthorized access to secrets                        |
| **Limit access to AWSCURRENT only**                       | Use `secretsmanager:VersionStage` condition key              |
| **Use tags for access control**                           | Tag secrets by environment, application, or team             |
| **Do not use long-term credentials with Secrets Manager** | Use IAM roles for applications and EC2 instance profiles     |
| **Replicate secrets across regions for DR**               | Ensure secrets are available in secondary regions            |
| **Use alternating users rotation strategy**               | Zero-downtime rotation for production databases              |
| **Delete secrets properly**                               | Use scheduled deletion (7-30 days) to allow recovery         |
| **Integrate with CloudFormation**                         | Deploy and manage secrets as infrastructure as code          |

## 12. Common Exam Scenarios

1. **Auto-rotation needed**: The scenario involves database credentials that must be rotated periodically → answer is **Secrets Manager** (not SSM Parameter Store).

2. **Rotation without downtime**: Application cannot tolerate a brief window where credentials are invalid → use the **alternating users (dual-user)** rotation strategy.

3. **Cross-account secret access**: Account B needs to read a secret in Account A → add a **resource-based policy** to the secret in Account A granting access to Account B.

4. **Limiting access to current secret only**: An IAM role should only be able to read the current version of a secret → use `secretsmanager:VersionStage` condition key set to `AWSCURRENT`.

5. **Secrets Manager vs Parameter Store**: Question asks for the cheapest option to store a password that does NOT need rotation → SSM Parameter Store (Standard, SecureString, free).

6. **Question asks for the best service to store an API key that needs rotation every 30 days** → Secrets Manager.

7. **Multi-region secret**: Global app needs the same credentials across regions → create a primary secret + replicas in other regions.

8. **Custom Lambda rotation**: Rotating secrets for a non-RDS service (third-party API, custom app) → **write a custom Lambda rotation function**.

9. **Secret recovery**: A secret was deleted 20 days ago — can it be recovered? → yes, within the 30-day pending deletion window, use `RestoreSecret`.

10. **KMS key for Secrets Manager**: Need to audit when secrets are accessed → use a **customer managed KMS key** (CloudTrail logs KMS Decrypt calls for each secret retrieval).

## 13. Exam Tips

1. **Primary differentiator**: Secrets Manager supports **automatic rotation**. SSM Parameter Store does not. This is the #1 exam differentiator.

2. **Staging labels**: `AWSCURRENT` = active, `AWSPREVIOUS` = last version, `AWSPENDING` = next version (during rotation).

3. **Rotation strategies**: Single user (brief downtime risk) vs Alternating users (zero downtime — preferred for production).

4. **Rotation Lambda**: AWS provides managed templates for RDS, Redshift, DocumentDB. For other services, write a custom Lambda.

5. **Encryption is always on**: Secrets are always encrypted at rest with KMS — unlike SSM Parameter Store where SecureString is optional.

6. **Max secret size**: 64 KB total; 10 KB for JSON via GetSecretValue (but can retrieve larger secrets with the full response).

7. **Cross-account**: Use resource-based policies on secrets. SSM Parameter Store does not natively support this.

8. **KMS key**: Default is `aws/secretsmanager`. Use a **customer managed key** for separate permissions and auditing.

9. **Deletion**: 30-day pending deletion window (configurable 7-30 days). Use `RestoreSecret` to recover during this window.

10. **Multi-Region**: Create primary + replicas. Replicas are read-only; rotation happens on the primary.

11. **CloudTrail**: Logs all `GetSecretValue` calls — critical for detecting unauthorized secret access.

12. **Cost**: $0.40/secret/month + $0.05/10k API calls. More expensive than SSM Parameter Store.

13. **EventBridge**: Rotation lifecycle events (`RotationStarted`, `RotationSucceeded`, `RotationFailed`) for monitoring.

14. **Key condition**: `secretsmanager:VersionStage` — restricts access to specific staging labels (e.g., only AWSCURRENT).

15. **Secrets Manager is regional**: Secrets are stored in a single region by default. Use Multi-Region replicas for DR.

16. **Tags**: Use tags (`secretsmanager:ResourceTag/<key>`) for fine-grained access control.

17. **ECS/EKS integration**: Secrets can be injected as environment variables or mounted as volumes in container environments.

18. **RDS password rotation**: Secrets Manager changes the password in both the secret AND the database — seamless.

19. **ForceDeleteWithoutRecovery**: Skips the waiting period. Use only when you are certain the secret is no longer needed.

20. **When rotation fails**: The secret stays at `AWSCURRENT` (old value is preserved). `AWSPENDING` is discarded. CloudWatch alarm on `RotationFailed`.
