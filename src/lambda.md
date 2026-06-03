# AWS Lambda

<figure>
  <img src="./images/Arch_AWS-Lambda_64@5x.png" alt="AWS Lambda Icon" width=200>
  <figcaption><center>AWS Lambda<br><i>Image source: AWS Documentation</i></center></figcaption>
</figure>

**Overview**: AWS Lambda is a serverless compute service that runs code in response to events and automatically manages the underlying compute resources. On the Security Specialty exam, Lambda is primarily tested in **incident response scenarios** (automated remediation), **security automation** (guardrails, compliance checks), and **access control** (execution roles, resource-based policies, VPC configuration).

**Domain weight**: Appears across Infrastructure Security (~17%), Detection & Logging (~18%), and Incident Response (~19%). Lambda itself is not a security service, but it is the **primary compute engine for security automation** on AWS.

## 1. Lambda Execution Role (IAM Role)

### 1.1. Purpose

- Every Lambda function has an **execution role** (an IAM role) that defines what AWS resources the function can access
- Lambda **assumes** this role when it executes — the role's permissions determine what the function can do
- The execution role is **not** tied to a specific user — it is a service role assumed by Lambda
- Lambda adds `sts:AssumeRole` trust policy for `lambda.amazonaws.com` automatically

### 1.2. Minimum Required Permissions

| Permission                      | Purpose                                             |
| ------------------------------- | --------------------------------------------------- |
| `logs:CreateLogGroup`           | Create a CloudWatch Logs log group for the function |
| `logs:CreateLogStream`          | Create a log stream within the log group            |
| `logs:PutLogEvents`             | Write log events to CloudWatch Logs                 |
| `ec2:CreateNetworkInterface`    | Required if the function is attached to a VPC       |
| `ec2:DescribeNetworkInterfaces` | Required if the function is attached to a VPC       |
| `ec2:DeleteNetworkInterface`    | Required if the function is attached to a VPC       |

### 1.3. Execution Role vs User Role

| Role Type               | When Used                                        | Created By    | Trusts                                  |
| ----------------------- | ------------------------------------------------ | ------------- | --------------------------------------- |
| **Execution role**      | When the function runs                           | Account admin | Lambda service (`lambda.amazonaws.com`) |
| **User/developer role** | When creating/updating functions via API/console | IAM admin     | IAM users or federated identities       |

**Exam scenario**: A Lambda function needs to read from an S3 bucket and write to DynamoDB → the Lambda **execution role** needs `s3:GetObject` and `dynamodb:PutItem` permissions.

## 2. Resource-Based Policies (Function Policy)

### 2.1. Purpose

- A **resource-based policy** attached directly to the Lambda function
- Controls which **other AWS accounts** or **AWS services** can invoke the function
- Also known as the **function policy**
- Used for **cross-account invocation** — allows Account B to invoke a Lambda function in Account A
- Max policy size: 20 KB

### 2.2. Common Use Cases

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:MyFunction",
      "Condition": {
        "ArnLike": {
          "AWS:SourceArn": "arn:aws:s3:::my-bucket"
        }
      }
    }
  ]
}
```

### 2.3. Who Can Invoke a Lambda Function

| Invoker            | Mechanism                                                                                                     |
| ------------------ | ------------------------------------------------------------------------------------------------------------- |
| **Same account**   | IAM policy on the caller's role/user grants `lambda:InvokeFunction`                                           |
| **Cross-account**  | Resource-based policy on the Lambda function grants access to the external account + IAM policy on the caller |
| **AWS service**    | Resource-based policy with `Principal": {"Service": "..."}`                                                   |
| **Another Lambda** | Resource-based policy or the caller's IAM role must have `lambda:InvokeFunction`                              |

**Exam scenario**: Account A has a Lambda function that Account B needs to invoke → Account A adds a **resource-based policy** on the Lambda function granting Account B's root user (or specific IAM entities) `lambda:InvokeFunction`. Account B also needs an IAM policy allowing the invoke action.

## 3. Event Source Mapping

### 3.1. Synchronous vs Asynchronous Invocation

| Invocation Type  | Description                                                      | Source Examples                                       |
| ---------------- | ---------------------------------------------------------------- | ----------------------------------------------------- |
| **Synchronous**  | Caller waits for the function to complete and returns the result | API Gateway, Cognito, Lex, Alexa, Lambda Function URL |
| **Asynchronous** | Caller queues the event; Lambda retries up to 2 times by default | S3, SNS, CloudWatch Events, EventBridge               |
| **Poll-based**   | Lambda polls a source and invokes the function with batches      | SQS, DynamoDB Streams, Kinesis                        |

### 3.2. Poll-Based Event Sources (Streams & Queues)

| Source               | Batch Size | Polling Details                          |
| -------------------- | ---------- | ---------------------------------------- |
| **SQS**              | 1–10       | Long polling; deletes message on success |
| **DynamoDB Streams** | 1–1000     | Iterator-based; checkpoints per shard    |
| **Kinesis**          | 1–10000    | Iterator-based; checkpoints per shard    |

### 3.3. Event Source Mapping Permissions

For poll-based sources, the **execution role** needs:

- **SQS**: `sqs:ReceiveMessage`, `sqs:DeleteMessage`, `sqs:GetQueueAttributes`
- **DynamoDB Streams**: `dynamodb:DescribeStream`, `dynamodb:GetRecords`, `dynamodb:GetShardIterator`
- **Kinesis**: `kinesis:DescribeStream`, `kinesis:GetRecords`, `kinesis:GetShardIterator`

**Exam scenario**: A Lambda function processes messages from an SQS queue. After processing, the message should be removed from the queue → the execution role needs `sqs:ReceiveMessage` AND `sqs:DeleteMessage` permissions. Without `DeleteMessage`, the message will be reprocessed (since the visibility timeout will expire).

## 4. Lambda in a VPC

### 4.1. How It Works

- By default, Lambda runs in an **AWS-managed VPC** (no VPC access)
- When you attach a Lambda function to a VPC, Lambda creates an **ENI** (Elastic Network Interface) in each subnet specified
- The ENI is created using `AWSLambdaVPCAccessExecutionRole` (managed policy)
- The function can access resources inside the VPC (RDS, ElastiCache, internal ALB, etc.)
- **No public internet access** when in a VPC — unless you add a NAT Gateway or VPC endpoints

### 4.2. VPC Permissions

```json
{
  "Effect": "Allow",
  "Action": [
    "ec2:CreateNetworkInterface",
    "ec2:DescribeNetworkInterfaces",
    "ec2:DeleteNetworkInterface",
    "ec2:AssignPrivateIpAddresses",
    "ec2:UnassignPrivateIpAddresses"
  ],
  "Resource": "*"
}
```

### 4.3. Internet Access from VPC

| Network Setup                   | Internet Access?           | Use Case                             |
| ------------------------------- | -------------------------- | ------------------------------------ |
| Lambda in VPC (private subnets) | ❌ No                      | Internal resources only              |
| Lambda in VPC + NAT Gateway     | ✅ Yes                     | Access internet + internal resources |
| Lambda in VPC + VPC Endpoints   | ✅ Yes (specific services) | Access AWS services privately        |
| Lambda outside VPC (default)    | ✅ Yes                     | AWS APIs, public endpoints           |

**Exam scenario**: A Lambda function needs to access an RDS database in a private subnet → attach the Lambda function to the **same VPC and subnets** as the RDS instance. The Lambda execution role needs `ec2:CreateNetworkInterface` permission.

**Exam scenario**: A Lambda function in a VPC needs to call the AWS KMS API → create a **VPC Endpoint** for KMS (or use a NAT Gateway) so the function can reach KMS without internet access.

### 4.4. VPC Endpoints for Lambda

- Use **Interface VPC Endpoints (AWS PrivateLink)** for Lambda API calls from within a VPC
- Endpoint: `com.amazonaws.<region>.lambda`
- Allows Lambda functions in a VPC to invoke other Lambda functions or manage Lambda resources without NAT Gateway
- Bucket policy-style condition keys: `aws:SourceVpce` and `aws:SourceVpc`

## 5. Lambda Environment Variables

### 5.1. Encryption

- Environment variables can be encrypted at rest using **KMS**
- **Default**: AWS managed key (`aws/lambda`) encrypts environment variables at rest
- **Custom**: Use a customer managed KMS key for encryption
- When the function is invoked, Lambda **automatically decrypts** the environment variables and makes them available to the function code

### 5.2. KMS Permission Requirements

```json
{
  "Effect": "Allow",
  "Action": [
    "kms:Encrypt",
    "kms:Decrypt",
    "kms:GenerateDataKey",
    "kms:ReEncrypt"
  ],
  "Resource": "arn:aws:kms:us-east-1:123456789012:key/your-key-id"
}
```

- If using a customer managed key, the Lambda **execution role** must have `kms:Decrypt` permission
- If using the default `aws/lambda` key, Lambda manages this automatically
- Environment variables encrypted with a customer managed KMS key must have the key policy grant access to the Lambda service

**Exam scenario**: A Lambda function stores database credentials in environment variables → use **encrypted environment variables** with a customer managed KMS key for separate permissions and audit trail.

**Important exam warning**: Environment variables are visible in the Lambda console and API responses (encrypted, but anyone with `lambda:GetFunctionConfiguration` can see the encrypted value). For production secrets, use **Secrets Manager** or **Parameter Store** instead.

## 6. Lambda Secrets Management (Exam Critical)

### 6.1. Secrets Manager Integration

- Lambda can retrieve secrets from **Secrets Manager** at runtime
- The execution role needs `secretsmanager:GetSecretValue`
- Best practice: retrieve the secret at **function initialization** (outside the handler) — it is cached for the lifetime of the execution environment
- Secret is decrypted server-side by Secrets Manager and returned over TLS

### 6.2. Parameter Store Integration

- Lambda can retrieve parameters from **SSM Parameter Store** at runtime
- The execution role needs `ssm:GetParameter` (for plaintext) or `ssm:GetParameter` with `WithDecryption=True` (for SecureString)
- Same caching best practice as Secrets Manager

### 6.3. Comparison

| Method                                | Use Case                                  | Limitation                                |
| ------------------------------------- | ----------------------------------------- | ----------------------------------------- |
| **Environment variables**             | Simple, static config (non-secret)        | Visible in console; max 4 KB              |
| **Encrypted env vars**                | Secrets with KMS encryption (basic)       | Need KMS permissions; visible (encrypted) |
| **Secrets Manager**                   | Secrets needing auto-rotation             | More expensive; additional latency        |
| **Parameter Store SecureString**      | Secrets without rotation, cost-sensitive  | No rotation; max 8 KB (Advanced)          |
| **Secrets Manager + Lambda rotation** | Auto-rotate secrets using a custom Lambda | Custom Lambda function required           |

**Exam scenario**: A Lambda function needs database credentials that are rotated every 30 days → retrieve the credentials from **Secrets Manager** at runtime. Use Secrets Manager's automatic rotation with a Lambda rotation function.

## 7. Lambda Code Signing

### 7.1. Purpose

- Ensures that only **signed and trusted code** can be deployed to Lambda
- Uses **AWS Signer** to sign Lambda function code
- Code signing verifies: the code has not been tampered with, the code was signed by an approved signer
- The code signing configuration is applied at the **function** or **layer** level
- Can be enforced at the **account level** via IAM policies or SCPs

### 7.2. How It Works

1. Developer signs the deployment package using AWS Signer (or an external signer)
2. The code signing configuration specifies **trusted signing profiles**
3. When you deploy the signed package, Lambda verifies:
   - The signature is valid
   - The signing profile is in the trusted list
   - The code has not been modified since signing
4. If validation fails, the deployment is **rejected**

**Exam scenario**: A security policy requires that all Lambda functions must only run approved code that has been signed by a trusted source → configure **Lambda code signing** with approved signing profiles.

## 8. Lambda@Edge

### 8.1. What It Is

- Lambda functions that run at **CloudFront edge locations**
- Used to customize content delivered through CloudFront
- Triggers: **Viewer Request**, **Origin Request**, **Origin Response**, **Viewer Response**
- Code runs in the region closest to the user (not in your primary region)

### 8.2. Security Considerations

| Aspect                    | Lambda@Edge                                           | Regular Lambda                  |
| ------------------------- | ----------------------------------------------------- | ------------------------------- |
| **Execution location**    | CloudFront edge locations (global)                    | Your selected AWS Region(s)     |
| **Execution role**        | IAM role assumed at edge                              | IAM role assumed in your region |
| **VPC**                   | ❌ Cannot be attached to a VPC                        | ✅ Can be attached to a VPC     |
| **Max execution time**    | 5 seconds (viewer events), 30 seconds (origin events) | 15 minutes                      |
| **Environment variables** | ✅ Supported (max 10 KB total)                        | ✅ Supported                    |
| **Lambda Layers**         | ✅ Supported                                          | ✅ Supported                    |
| **Code size**             | 1 MB (viewer), 50 MB (origin)                         | 250 MB                          |
| **TLS termination**       | At edge (CloudFront)                                  | At the service endpoint         |

**Exam scenario**: A web application needs to add security headers to all HTTP responses at the edge, before responses reach users → use **Lambda@Edge** with a **Viewer Response** trigger to add headers like `Strict-Transport-Security`, `X-Content-Type-Options`, and `Content-Security-Policy`.

## 9. Lambda Function URLs

### 9.1. Overview

- Lambda functions can have a dedicated **HTTPS endpoint** (Function URL)
- Format: `https://<url-id>.lambda-url.<region>.on.aws/`
- **No API Gateway required** — direct invocation via HTTP
- Two auth types:

| Auth Type | Description                                                   |
| --------- | ------------------------------------------------------------- |
| `AWS_IAM` | Requests must be signed with SigV4 (IAM authentication)       |
| `NONE`    | Public endpoint — anyone with the URL can invoke the function |

### 9.2. Security Implications

- **`AWS_IAM`** is recommended for production — ensures only authenticated principals can invoke the function
- **`NONE`** is for public-facing functions — the function itself must handle authorization (e.g., validate JWT tokens)
- The resource-based policy on the function controls who can invoke it via the URL
- Cross-Origin Resource Sharing (CORS) can be configured on Function URLs

**Exam scenario**: A Lambda function needs a public HTTPS endpoint without managing API Gateway → use a **Lambda Function URL** with `AWS_IAM` auth type.

## 10. Lambda Layers

### 10.1. Security Considerations

- A Lambda layer is a ZIP archive containing libraries, custom runtimes, or other dependencies
- Layers are **shared across functions** — a single layer can be used by many functions
- **Layer permissions** control which accounts or OUs can use the layer:
  - Private (default): only the owner account can use it
  - Public: anyone can use it (risky)
  - Specific accounts: only specified AWS accounts can use it
- Layers must be from **trusted sources** — a compromised layer affects all functions using it

**Exam scenario**: An organization needs to share a common security library across all Lambda functions in the organization → create a **Lambda Layer** with the library and grant cross-account access to the layer for accounts in the organization.

## 11. Lambda Logging and Monitoring

### 11.1. CloudWatch Logs

- Lambda automatically sends logs to **CloudWatch Logs**
- The execution role needs `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`
- Log streams are named by function version and execution environment
- Logs include: START, END, REPORT lines (duration, billed duration, memory used, max memory used)
- Custom log output (via `console.log`, `print`, etc.) is also captured

### 11.2. CloudWatch Metrics

| Metric                 | Description                                         |
| ---------------------- | --------------------------------------------------- |
| `Invocations`          | Number of times the function is invoked             |
| `Errors`               | Number of failed invocations (exceptions, timeouts) |
| `Throttles`            | Number of times the function is throttled           |
| `Duration`             | Execution time in milliseconds                      |
| `ConcurrentExecutions` | Number of concurrent executions                     |
| `DeadLetterErrors`     | Number of errors when sending events to DLQ         |

### 11.3. AWS X-Ray

- Lambda integrates with **AWS X-Ray** for tracing
- Tracks requests as they flow through Lambda and downstream services
- Can identify performance bottlenecks and errors
- The execution role needs `xray:PutTraceSegments` and `xray:PutTelemetryRecords`
- **Active tracing** must be enabled on the function

### 11.4. CloudTrail

- Logs Lambda **management API calls**: `CreateFunction`, `UpdateFunctionCode`, `InvokeFunction`, etc.
- **Data events** (`InvokeFunction`) can be logged but are **expensive** at scale
- CloudTrail does **not** log the function's internal execution — use CloudWatch Logs for that

## 12. Lambda Concurrency and Throttling

### 12.1. Concurrency Limits

| Limit Type                  | Value                               | Notes                                      |
| --------------------------- | ----------------------------------- | ------------------------------------------ |
| **Account-level**           | 1000 concurrent executions          | Per region, per account (can be increased) |
| **Function-level**          | Can be set lower than account limit | **Reserved concurrency** limits a function |
| **Provisioned concurrency** | Pre-warms execution environments    | Reduces cold start latency                 |

### 12.2. Security Implications

- **Reserved concurrency**: Ensures a critical security function always has capacity — limits other functions from using all the account concurrency
- **Provisioned concurrency**: Can be used for latency-sensitive security automation
- **Throttling**: When concurrency is exhausted, Lambda returns `429 TooManyRequestsException`

**Exam scenario**: A security-critical Lambda function (e.g., automated incident response) must always have capacity to execute, even under high load → configure **reserved concurrency** on the function.

## 13. Lambda Dead Letter Queue (DLQ)

### 13.1. Purpose

- For **asynchronous** invocations, if the function fails (returns an error), the event can be sent to a **DLQ**
- DLQ destinations: **SQS queue** or **SNS topic**
- Up to **2 automatic retries** before sending to DLQ
- Can also configure **on-failure destinations** (SQS, SNS, Lambda, EventBridge)

### 13.2. DLQ vs On-Failure Destination

| Feature           | Dead Letter Queue (DLQ) | On-Failure Destination                                |
| ----------------- | ----------------------- | ----------------------------------------------------- |
| **Destinations**  | SQS, SNS                | SQS, SNS, Lambda, EventBridge                         |
| **Configuration** | Legacy (function level) | Newer (per event source mapping)                      |
| **Flexibility**   | Limited                 | More flexible (different targets for success/failure) |

**Exam scenario**: A Lambda function processes events from S3. Processing fails, and after 2 retries, the event is lost → configure a **dead letter queue** (or on-failure destination) to capture failed events for reprocessing.

## 14. Lambda Best Practices for Security

| Practice                                                 | Description                                                    |
| -------------------------------------------------------- | -------------------------------------------------------------- |
| **Use least-privilege execution roles**                  | Grant only the specific permissions the function needs         |
| **Do not hardcode secrets**                              | Use Secrets Manager or Parameter Store at runtime              |
| **Encrypt environment variables**                        | Use a customer managed KMS key for sensitive configuration     |
| **Use Lambda layers for shared code**                    | Centrally manage and audit shared libraries                    |
| **Enable code signing**                                  | Ensure only trusted code is deployed                           |
| **Use reserved concurrency**                             | Prevent critical functions from being throttled                |
| **Monitor with CloudWatch Logs + Metrics**               | Detect errors, throttles, and unexpected behavior              |
| **Use VPC endpoints for AWS services**                   | Keep Lambda-to-AWS traffic within the AWS network              |
| **Use AWS X-Ray for tracing**                            | Monitor end-to-end request flow and identify security issues   |
| **Set function timeout appropriately**                   | Avoid long-running functions that could be exploited           |
| **Use resource-based policies for cross-account invoke** | Instead of sharing IAM roles                                   |
| **Validate and sanitize all inputs**                     | Lambda is exposed to untrusted input via API Gateway, S3, etc. |
| **Tag Lambda functions**                                 | Use tags for access control and cost allocation                |
| **Enable Lambda@Edge for edge security**                 | Add security headers, authenticate at the edge                 |
| **Use CloudFormation or SAM for deployment**             | Infrastructure as code for security-critical functions         |

## 15. Common Exam Scenarios

1. **Incident response automation**: A security alert triggers a Lambda function that isolates an EC2 instance, revokes IAM keys, or updates security groups → the **execution role** must have the specific permissions (e.g., `ec2:StopInstances`, `iam:UpdateAccessKey`).

2. **S3 event notification + Lambda**: An object is uploaded to S3, triggering a Lambda function that scans it for malware with Amazon GuardDuty or Macie → use **S3 Event Notification** with Lambda as the destination.

3. **Lambda in VPC + RDS**: A function needs to query a database -> attach the function to the VPC, ensure the execution role has `ec2:CreateNetworkInterface`, and the security group allows the connection.

4. **Cross-account Lambda invoke**: Account B's application needs to invoke a Lambda function in Account A → add a **resource-based policy** on the function in Account A.

5. **Secrets in Lambda**: Retrieving credentials from **Secrets Manager** at function initialization → the execution role needs `secretsmanager:GetSecretValue`.

6. **Lambda temp credentials via STS**: A Lambda function needs to assume a role in another account → the execution role must have `sts:AssumeRole` permission for the target role ARN.

7. **Lambda@Edge for web security**: Add security headers, rewrite URLs, or perform authentication at CloudFront edge locations → use **Lambda@Edge** with Viewer Request/Response triggers.

8. **GuardDuty + Lambda remediation**: GuardDuty finds a compromised EC2 instance → GuardDuty sends finding to EventBridge → EventBridge triggers a **Lambda function** that isolates the instance.

9. **Lambda function URL with IAM auth**: Expose a Lambda function via HTTPS without API Gateway → use **Function URL** with `AWS_IAM` auth type.

10. **Lambda code signing**: Prevent unauthorized code from being deployed → use **Lambda code signing** with AWS Signer.

## 16. Exam Tips

1. **Execution role is everything**: The Lambda execution role determines what the function can access. Exam scenarios often test which IAM permissions the execution role needs.

2. **VPC Lambda = no internet**: When you attach a Lambda to a VPC, it loses internet access unless you add a NAT Gateway or VPC Endpoints.

3. **ENI creation permissions**: Lambda in VPC needs `ec2:CreateNetworkInterface` in the execution role. This is a common distractor (wrong answer) — the exam expects you to know this.

4. **Resource-based policy for cross-account**: Same-account invocation uses IAM. Cross-account invocation needs a resource-based policy on the Lambda function.

5. **Secrets at runtime**: Retrieve secrets from Secrets Manager/Parameter Store at function **initialization** (outside the handler) — not on every invocation.

6. **KMS permissions for encrypted env vars**: If using a customer managed KMS key, the execution role needs `kms:Decrypt`.

7. **Lambda@Edge restrictions**: No VPC, limited execution time, limited code size. Cannot access resources in a VPC.

8. **Code signing**: Uses AWS Signer. Prevents tampered code from being deployed.

9. **Concurrency**: Reserved concurrency guarantees capacity for a function. Provisioned concurrency reduces cold starts.

10. **DLQ for async invocations**: After 2 retries, failed events go to DLQ (SQS or SNS).

11. **Lambda Layers**: Can be shared across accounts. Be careful with public layers.

12. **CloudTrail vs CloudWatch**: CloudTrail logs Lambda API calls (InvokeFunction, CreateFunction). CloudWatch Logs captures function execution output.

13. **Function URLs**: `AWS_IAM` auth type requires SigV4 signing. `NONE` auth type is public.

14. **Lambda timeouts**: Max 15 minutes. Security automation functions should have appropriate timeouts.

15. **Event source mappings**: For poll-based sources (SQS, Kinesis, DynamoDB Streams), the execution role needs source-specific permissions (ReceiveMessage, DeleteMessage, etc.).

16. **Lambda + Config**: Lambda can be used as an AWS Config remediation action or custom rule.

17. **Lambda + Security Hub**: Lambda can process Security Hub findings for automated remediation.

18. **Lambda + GuardDuty**: Lambda is the compute engine for GuardDuty automated response (via EventBridge).

19. **Lambda environment re-use**: The execution environment is re-used for subsequent invocations — cache connections and SDK clients outside the handler.

20. **Lambda in an organization**: Use **resource-based policies** for cross-account layer sharing. Use **SCPs** to enforce Lambda security controls (e.g., require VPC config, require code signing).
