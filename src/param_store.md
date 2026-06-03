# AWS Systems Manager Parameter Store

<figure>
  <img src="./images/Arch_AWS-Systems-Manager_64@5x.png" alt="AWS Systems Manager Parameter Store Icon" width=200>
  <figcaption><center>AWS Systems Manager Parameter Store<br><i>Image source: AWS Documentation</i></center></figcaption>
</figure>

**Overview**: AWS Systems Manager Parameter Store is a fully managed service for storing configuration data and secrets as **parameters**. It provides a secure, hierarchical store for values such as database connection strings, license codes, AMI IDs, passwords, and API keys. It is often compared to AWS Secrets Manager on the exam — Parameter Store is the simpler, cheaper option but lacks built-in automatic rotation.

**Domain weight**: Appears across Data Protection (~18%) and IAM (~20%) domains. The exam primarily tests Parameter Store via comparison questions with Secrets Manager and through its integration with other AWS services (CloudFormation `resolve:ssm`, EC2 Run Command, Lambda).

**Key relationship**: Parameter Store is part of **AWS Systems Manager (SSM)** — not a standalone service. It is accessed via the SSM API (`ssm:GetParameter`, `ssm:PutParameter`, etc.).

## 1. Parameter Tiers

| Feature                    | Standard Tier                 | Advanced Tier                  |
| -------------------------- | ----------------------------- | ------------------------------ |
| **Cost**                   | Free                          | $0.05 per parameter per month  |
| **Max parameter size**     | 4 KB                          | 8 KB                           |
| **Max parameters**         | 10,000 per account per region | 100,000 per account per region |
| **Parameter policies**     | Not supported                 | ✅ Supported                   |
| **Versioning**             | ✅ Yes                        | ✅ Yes                         |
| **KMS encryption**         | ✅ Yes (SecureString)         | ✅ Yes (SecureString)          |
| **Allowed characters**     | 4 KB of UTF-8                 | 8 KB of UTF-8                  |
| **CloudFormation support** | ✅ Yes                        | ✅ Yes                         |
| **Tagging**                | ✅ Yes                        | ✅ Yes                         |

- **Standard**: Sufficient for most use cases. Free, 4 KB limit, no parameter policies.
- **Advanced**: For larger values (up to 8 KB) or when you need parameter policies (expiration, change notifications).

**Exam scenario**: A configuration value is 6 KB and needs to be stored in Parameter Store → use the **Advanced** tier (Standard tier max is 4 KB).

## 2. Parameter Types

| Type             | Description                                                   | Max Size    |
| ---------------- | ------------------------------------------------------------- | ----------- |
| **String**       | Plaintext string (e.g., URL, AMI ID, feature flag)            | 4 KB / 8 KB |
| **StringList**   | Comma-separated list of values (e.g., `value1,value2,value3`) | 4 KB / 8 KB |
| **SecureString** | Encrypted value using AWS KMS (SSE-KMS)                       | 4 KB / 8 KB |

### 2.1. String vs StringList

- **String**: A single text value. Supports hierarchies (`/prod/db/host`).
- **StringList**: A comma-delimited list. Each element is a value, not a key-value pair.

### 2.2. SecureString (Encrypted Parameters)

- Uses **AWS KMS** to encrypt the parameter value at rest
- By default, uses the **AWS managed key** `aws/ssm`
- Can use a **customer managed KMS key** for separate permissions and audit trail
- When retrieved, the value is **decrypted server-side** and returned over TLS (HTTPS)
- To retrieve: `GetParameter` with `WithDecryption=True`
- To create: `PutParameter` with `Type=SecureString`

**Exam scenario**: An application needs to store a database password in Parameter Store → use **SecureString** parameter type with a customer managed KMS key for encryption at rest.

**Key exam point**: If you do NOT need automatic rotation, **SecureString in Parameter Store** is the cheaper alternative to Secrets Manager for storing secrets.

## 3. Parameter Hierarchies

- Parameters can be organized using a **path-like naming convention**:
  - `/prod/db/host`
  - `/prod/db/port`
  - `/prod/db/username`
  - `/prod/db/password`
  - `/staging/db/host`
  - `/dev/db/host`
- Hierarchies enable **bulk retrieval**: `GetParametersByPath` returns all parameters under a path
- Supports **wildcard patterns** in IAM policies using `ssm:ResourceTag/<key>` and `aws:ResourceTag`

**Exam scenario**: An application needs to retrieve all database configuration parameters in one API call → store parameters in a hierarchy (e.g., `/prod/db/`) and use `GetParametersByPath` to retrieve them all.

## 4. Versioning

- Each parameter update creates a **new version** (sequential integer: 1, 2, 3, ...)
- Old versions are **retained** and can be retrieved by version number
- Default: `GetParameter` returns the **latest** version
- To retrieve a specific version: specify the `Version` parameter
- Version history is **retained indefinitely** (unlike Secrets Manager's staging label approach)
- `GetParameterHistory` returns all versions of a parameter with metadata

**Exam scenario**: A bad configuration change was made, and the application needs to roll back to a previous value → use `GetParameter` with the previous version number, or use `PutParameter` to overwrite with the older value.

## 5. Parameter Policies (Advanced Tier Only)

| Policy Type                | Description                                                                                    |
| -------------------------- | ---------------------------------------------------------------------------------------------- |
| **Expiration**             | Deletes the parameter at a specified date/time                                                 |
| **ExpirationNotification** | Sends an EventBridge event before expiration (up to 30 days advance)                           |
| **NoChangeNotification**   | Sends an EventBridge event if the parameter has not been updated in a specified number of days |

- Only available for **Advanced tier** parameters
- Configurable via the `Policies` parameter in `PutParameter`
- Useful for managing temporary credentials, API keys, or time-limited configuration
- Events are sent to **Amazon EventBridge** — can trigger Lambda, SNS, SQS, etc.

**Exam scenario**: An API key stored in Parameter Store must be automatically removed after a specific date → use an **Advanced tier** parameter with an **Expiration policy**.

## 6. Encryption (SecureString)

### 6.1. KMS Key Options

| Key Type                 | Description                                                         |
| ------------------------ | ------------------------------------------------------------------- |
| **AWS managed key**      | `aws/ssm` — free, no separate permissions, no rotation control      |
| **Customer managed CMK** | You create and manage — separate permissions, rotation, audit trail |

### 6.2. How Encryption Works

1. Parameter Store uses **envelope encryption** via KMS
2. When `PutParameter` with `Type=SecureString` is called:
   - Parameter Store calls KMS `GenerateDataKey` (or `Encrypt`) with the specified KMS key
   - The parameter value is encrypted with the data key
   - The encrypted value is stored in Parameter Store
3. When `GetParameter` with `WithDecryption=True` is called:
   - Parameter Store calls KMS `Decrypt` with the encrypted data key
   - The parameter value is decrypted and returned over TLS

**Exam scenario**: A compliance requirement mandates that access to encrypted parameters be auditable via CloudTrail → use a **customer managed KMS key** (CloudTrail logs each KMS `Decrypt` call), and monitor with `aws/ssm` key audit.

## 7. Access Control

### 7.1. IAM Actions

| Action                      | Description                                          |
| --------------------------- | ---------------------------------------------------- |
| `ssm:GetParameter`          | Retrieve a single parameter (value may be encrypted) |
| `ssm:GetParameters`         | Retrieve multiple parameters by name                 |
| `ssm:GetParametersByPath`   | Retrieve all parameters under a hierarchy path       |
| `ssm:PutParameter`          | Create or update a parameter                         |
| `ssm:DeleteParameter`       | Delete a parameter                                   |
| `ssm:DescribeParameters`    | List metadata about parameters (not values)          |
| `ssm:GetParameterHistory`   | Retrieve version history of a parameter              |
| `ssm:LabelParameterVersion` | Add/remove labels from parameter versions            |

### 7.2. Condition Keys

| Condition Key           | Description                                       |
| ----------------------- | ------------------------------------------------- |
| `ssm:ResourceTag/<key>` | Restrict based on parameter tags                  |
| `aws:ResourceTag/<key>` | Global condition for resource tags                |
| `aws:SourceIp`          | Restrict by IP address                            |
| `aws:SourceVpc`         | Restrict by VPC                                   |
| `aws:SourceVpce`        | Restrict by VPC Endpoint                          |
| `aws:SecureTransport`   | Require HTTPS                                     |
| `ssm:Overwrite`         | Allows PutParameter only if overwrite flag is set |

**Exam tip**: Unlike Secrets Manager, Parameter Store does **not** support resource-based policies — you cannot attach a policy directly to a parameter. Cross-account access requires using IAM role assumption (STS `AssumeRole`).

### 7.3. IAM Policy Example (Hierarchy-Based Access)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ssm:GetParameter",
      "Resource": "arn:aws:ssm:us-east-1:123456789012:parameter/prod/db/*"
    }
  ]
}
```

This grants access to all parameters under the `/prod/db/` hierarchy.

## 8. Parameter Store vs Secrets Manager

This is one of the most frequently tested comparison topics on the exam.

| Feature                       | Parameter Store                              | Secrets Manager                           |
| ----------------------------- | -------------------------------------------- | ----------------------------------------- |
| **Purpose**                   | Configuration data + secrets                 | Secrets only                              |
| **Max value size**            | 4 KB (Standard), 8 KB (Advanced)             | 64 KB (10 KB for JSON via GetSecretValue) |
| **Encryption**                | Optional (SecureString with KMS)             | Always encrypted with KMS                 |
| **Automatic rotation**        | ❌ Not supported                             | ✅ Built-in with Lambda                   |
| **Rotation Lambda templates** | ❌ N/A                                       | ✅ RDS, Redshift, DocumentDB              |
| **Cross-account access**      | ❌ Via IAM role assumption only              | ✅ Resource-based policies                |
| **Versioning**                | Sequential version numbers                   | Staging labels (AWSCURRENT, AWSPREVIOUS)  |
| **Cost**                      | Standard: **Free**; Advanced: $0.05/param/mo | $0.40/secret/month + $0.05/10k API calls  |
| **Parameter policies**        | Advanced tier only                           | ❌ N/A (replaced by rotation config)      |
| **Hierarchies**               | ✅ Yes (path-based names)                    | ❌ Not supported                          |
| **Bulk retrieval**            | ✅ GetParametersByPath                       | ❌ Must list + retrieve individually      |
| **Resource-based policies**   | ❌ Not supported                             | ✅ Supported                              |
| **KMS integration**           | Optional (SecureString only)                 | Mandatory (always encrypted)              |
| **CloudFormation resolve**    | ✅ `{{resolve:ssm:...}}`                     | ✅ `{{resolve:secretsmanager:...}}`       |
| **Tag-based access control**  | ✅ Yes                                       | ✅ Yes                                    |

### 8.1. When to Use Which (Exam-Focused)

| Scenario                                                         | Choose                             |
| ---------------------------------------------------------------- | ---------------------------------- |
| Store database password with auto-rotation                       | **Secrets Manager**                |
| Store database password without rotation (cost-sensitive)        | **Parameter Store (SecureString)** |
| Store non-sensitive config (URLs, feature flags, AMI IDs)        | **Parameter Store (String)**       |
| Store secrets that need cross-account access                     | **Secrets Manager**                |
| Store large secrets (up to 64 KB)                                | **Secrets Manager**                |
| Store config up to 4 KB for free                                 | **Parameter Store (Standard)**     |
| Organize config hierarchically (`/prod/db/host`, `/dev/db/host`) | **Parameter Store**                |
| Need automatic secret rotation for RDS                           | **Secrets Manager**                |
| Need parameter expiration or change notifications                | **Parameter Store (Advanced)**     |

**Exam scenario**: An application needs to store database credentials and rotate them monthly. Cost is not the primary concern → use **AWS Secrets Manager**.

**Exam scenario**: An application needs to store database credentials and rotate them monthly, but cost must be minimized → this is a trick question — Parameter Store does not support rotation. Secrets Manager is the only option that meets the rotation requirement.

**Exam scenario**: An application needs to store a simple database URL and port (non-secret, under 4 KB) → use **Parameter Store (Standard tier, String type)** — it's free.

## 9. Integration with AWS Services

| Service                | Integration                                                                       |
| ---------------------- | --------------------------------------------------------------------------------- |
| **AWS CloudFormation** | `{{resolve:ssm:/path/to/parameter}}` — reference parameter values in templates    |
| **AWS Lambda**         | Retrieve parameters within function code via SDK; inject as environment variables |
| **Amazon ECS/EKS**     | Inject parameters as environment variables or secrets in container definitions    |
| **EC2 Run Command**    | SSM Agent retrieves parameters for command execution                              |
| **EC2 State Manager**  | Use parameters in association configurations                                      |
| **AWS CodeBuild**      | Reference parameters in buildspec files                                           |
| **AWS CodePipeline**   | Use parameters in pipeline actions                                                |
| **Amazon EventBridge** | Parameter policies trigger events (Advanced tier)                                 |
| **AWS CloudTrail**     | Logs all Parameter Store API calls                                                |
| **AWS Config**         | Track parameter configuration changes                                             |

### 9.1. CloudFormation Dynamic References

```yaml
Resources:
  MyRDSInstance:
    Type: AWS::RDS::DBInstance
    Properties:
      MasterUsername: "{{resolve:ssm:/prod/db/username}}"
      MasterUserPassword: "{{resolve:ssm-secure:/prod/db/password}}"
```

- `{{resolve:ssm:...}}` — references plaintext parameters (String, StringList)
- `{{resolve:ssm-secure:...}}` — references encrypted parameters (SecureString), decrypts them during stack creation
- CloudFormation supports parameter **version** references: `{{resolve:ssm:/param:version}}`

**Exam scenario**: A CloudFormation template needs to reference a database password stored in Parameter Store without exposing it in the template → use **dynamic references** with `{{resolve:ssm-secure:/path/to/secure-parameter}}`.

## 10. Monitoring and Auditing

### 10.1. CloudTrail

- All Parameter Store API calls are logged in CloudTrail
- Key events to monitor:
  - `GetParameter` with `WithDecryption=True` (decrypted secret access)
  - `PutParameter` (parameter created/updated)
  - `DeleteParameter` (parameter deleted)
  - `LabelParameterVersion` (version labeling)

### 10.2. CloudWatch Metrics

- Parameter Store emits metrics via **AWS Systems Manager**:
  - `GetParameter` calls
  - `PutParameter` calls
  - Throttled requests

### 10.3. EventBridge (Advanced Tier)

- Parameter policies generate EventBridge events:
  - **ExpirationNotification**: Parameter is about to expire
  - **NoChangeNotification**: Parameter has not been updated in X days

### 10.4. AWS Config

- Use AWS Config rules to track Parameter Store resource changes:
  - Custom rules can validate parameter naming conventions
  - Check that certain parameters are tagged correctly
  - Monitor for SecureString parameter creation without customer managed KMS key

## 11. Cost

| Tier      | Cost                                                   |
| --------- | ------------------------------------------------------ |
| Standard  | **Free** (up to 10,000 parameters)                     |
| Advanced  | $0.05 per parameter per month                          |
| API calls | Free (Standard); $0.05 per 10,000 API calls (Advanced) |

**Exam tip**: Parameter Store Standard tier is **free** — this is the primary reason to choose it over Secrets Manager when rotation is not needed.

## 12. Security Best Practices

| Practice                                   | Description                                                      |
| ------------------------------------------ | ---------------------------------------------------------------- |
| **Use SecureString for secrets**           | Encrypt passwords, API keys, tokens at rest                      |
| **Use customer managed KMS key**           | Separate permissions, rotation control, CloudTrail audit         |
| **Use parameter hierarchies**              | Organize by environment, application, or team                    |
| **Use IAM least privilege**                | Grant specific `ssm:GetParameter` access by path/hierarchy       |
| **Restrict access to WithDecryption=True** | Only grant decryption permission to roles that need it           |
| **Use CloudTrail monitoring**              | Detect unauthorized access to decrypted parameters               |
| **Use tags for access control**            | Tag parameters by environment, owner, sensitivity                |
| **Rotate SecureString values manually**    | Parameter Store does not auto-rotate — schedule periodic updates |
| **Use AWS Config rules**                   | Validate parameter naming, tagging, and KMS key usage            |
| **Use versioning for rollback**            | Revert to a previous parameter version if needed                 |
| **Do not store long-lived credentials**    | Prefer IAM roles over static credentials                         |

## 13. Limitations

| Limitation                           | Detail                                                     |
| ------------------------------------ | ---------------------------------------------------------- |
| **No automatic rotation**            | Must rotate secrets manually via script or Lambda          |
| **No resource-based policies**       | Cannot attach policy directly to a parameter               |
| **No cross-account access**          | Must use IAM role assumption (STS AssumeRole)              |
| **No native secret version labels**  | Uses sequential version numbers (no staging labels)        |
| **Max 4 KB (Standard tier)**         | Values larger than 4 KB require Advanced tier (up to 8 KB) |
| **Max 10,000 parameters (Standard)** | Limits may affect large deployments                        |
| **No integration with RDS**          | Cannot automatically update database passwords             |

## 14. Common Exam Scenarios

1. **Cheapest way to store a password**: If rotation is not required, **Parameter Store (SecureString, Standard tier)** is free and meets the requirement.

2. **Larger than 4 KB**: The config value exceeds 4 KB → use **Advanced tier** or **Secrets Manager**.

3. **Hierarchical organization**: Multiple config values need to be organized by environment → use **path-based names** (`/prod/db/url`, `/staging/db/url`) and `GetParametersByPath`.

4. **CloudFormation without exposing secrets**: Template needs to reference a database password → use `{{resolve:ssm-secure:/path/to/password}}`.

5. **Cross-account Parameter Store**: Account B needs Account A's parameter → Account A creates an **IAM role** (cannot use resource-based policy since Parameter Store does not support it). Account B assumes the role.

6. **Parameter Store vs Secrets Manager — rotation needed**: The exam asks for a solution that must rotate credentials → **Secrets Manager** is the only choice.

7. **Parameter Store vs Secrets Manager — cost priority**: The exam asks for the cheapest way to store a database password → **Parameter Store Standard tier** (free) if rotation is NOT required. If rotation IS required, Secrets Manager is the only option (cost is secondary).

8. **Parameter version rollback**: A bad config change broke the app → retrieve the previous version via `GetParameter` with the version number.

9. **Parameter expiration**: A temporary API key must be auto-deleted after a date → **Advanced tier** with **Expiration policy**.

10. **EC2 instance profile + Parameter Store**: An EC2 instance needs to retrieve a parameter → the instance profile role needs `ssm:GetParameter` permission.

## 15. Exam Tips

1. **Secrets Manager vs Parameter Store**: The exam WILL test this. Parameter Store = no rotation, cheaper. Secrets Manager = auto-rotation, more expensive.

2. **Standard tier is free**: Up to 10,000 parameters, 4 KB each, free of charge.

3. **SecureString uses KMS**: Default key is `aws/ssm`. Use a customer managed CMK for audit trails.

4. **No auto-rotation**: You must implement manual rotation via scripts, Lambda, or scheduled tasks.

5. **No resource-based policies**: This is a key limitation. Cannot attach a policy to a parameter — must use IAM roles for cross-account.

6. **Hierarchies**: Parameter Store supports path-based naming (`/env/app/key`). Secrets Manager does not.

7. **Bulk retrieval**: `GetParametersByPath` retrieves all parameters under a path. Secrets Manager has no equivalent.

8. **CloudFormation dynamic references**: `{{resolve:ssm:...}}` and `{{resolve:ssm-secure:...}}` — the secure variant decrypts the parameter.

9. **Versioning**: Sequential numbers, not staging labels. Use version numbers for rollback.

10. **Parameter policies**: Advanced tier only. Expiration, ExpirationNotification, NoChangeNotification.

11. **Max size**: Standard = 4 KB, Advanced = 8 KB. For larger values, use Secrets Manager (64 KB).

12. **Tags**: Use tags for access control and organization (`ssm:ResourceTag/<key>` condition).

13. **GetParameter vs GetParameters**: Use `GetParameters` to retrieve multiple named parameters in one call (up to 10 per call).

14. **CloudTrail**: Logs all API calls, including `GetParameter` with `WithDecryption=True` — monitor for unauthorized secret access.

15. **SecureString transmission**: The value is encrypted at rest in KMS, transmitted over TLS, but visible in the API response — ensure HTTPS is enforced.

16. **EC2 SSM Agent**: The SSM Agent can retrieve parameters for use in Run Command, State Manager, and other SSM capabilities.

17. **Parameter naming**: Names are case-sensitive. `/Prod/DB/Url` and `/prod/db/url` are different parameters.

18. **ARN format**: `arn:aws:ssm:<region>:<account>:parameter/<name>` — note the `parameter/` prefix before the name.

19. **Overwrite protection**: Use `PutParameter` with `Overwrite=False` to prevent accidental updates (returns error if parameter exists).

20. **LabelParameterVersion**: Advanced tier only — allows adding aliases like `latest`, `stable` to specific versions.
