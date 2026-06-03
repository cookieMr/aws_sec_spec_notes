# Amazon S3 Security Features

<figure>
  <img src="./images/Arch_Amazon-S3-on-Outposts_64@5x.png" alt="Amazon S3 Security Features Icon" width=200>
  <figcaption><center>Amazon S3 Security Features<br><i>Image source: AWS Documentation</i></center></figcaption>
</figure>

**Overview**: Amazon S3 (Simple Storage Service) is object storage with virtually unlimited scale. Security features span access control, encryption, data integrity, logging, and network restrictions. S3 is one of the most heavily tested services on the exam — expect 10–15+ questions involving S3 security across all domains.

**Domain weight**: S3 appears across Data Protection (~18%), Infrastructure Security (~17%), Detection & Logging (~18%), and IAM (~20%). It is the single most important service to study for the exam.

## 1. S3 Access Control

S3 has four access control mechanisms. Understanding which takes precedence is critical.

### 1.1. Access Control Mechanisms

| Mechanism                | Scope          | Type            | Evaluation Order |
| ------------------------ | -------------- | --------------- | ---------------- |
| **Bucket policy**        | Bucket         | Resource-based  | 1st              |
| **User policy (IAM)**    | User/Role      | Identity-based  | 2nd              |
| **Access Control Lists** | Bucket/Object  | Legacy ACL      | 3rd              |
| **Block Public Access**  | Bucket/Account | Override (Deny) | Always enforced  |

**Exam tip**: When evaluating access to an S3 object, AWS evaluates **all policies together**. An explicit deny in any policy type beats an allow in any other. Block Public Access is an **implicit override** — it acts as a blanket deny on all public access regardless of other policies.

### 1.2. Bucket Policies (Resource-Based)

- Attached directly to the S3 bucket (not to a user/role)
- Can grant access to other AWS accounts (cross-account access) — no need for IAM roles
- The `Principal` element is required (unlike IAM user policies)
- Maximum policy size: 20 KB
- Can use `aws:SourceIp`, `aws:SourceVpce`, `aws:SourceVpc`, `aws:SecureTransport`, and other condition keys

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-secure-bucket",
        "arn:aws:s3:::my-secure-bucket/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

This policy blocks all unencrypted (HTTP) requests to the bucket.

**Exam scenario**: You need to enforce HTTPS for all access to an S3 bucket → use bucket policy with `aws:SecureTransport: false` → Deny.

**Exam scenario**: Cross-account access to S3 → write a bucket policy granting the external account access, OR use an IAM role with cross-account trust. Bucket policy is simpler (no role assumption needed).

### 1.3. S3 Block Public Access

| Setting                               | Scope   | Effect                                                           |
| ------------------------------------- | ------- | ---------------------------------------------------------------- |
| `BlockPublicAcls`                     | Bucket  | Blocks ACLs that grant public access                             |
| `IgnorePublicAcls`                    | Bucket  | Ignores all public ACLs on the bucket and objects                |
| `BlockPublicPolicy`                   | Bucket  | Blocks bucket policies that grant public access                  |
| `RestrictPublicBuckets`               | Bucket  | Restricts public bucket policies to only AWS services/principals |
| **Account-level Block Public Access** | Account | Applied to ALL buckets in the account (cannot be overridden)     |

- Can be set at the **account level** (applies to all current and future buckets) and/or at the **bucket level**
- Account-level settings **override** bucket-level settings
- Once enabled, these settings act as an **explicit deny** for public access
- Even if a bucket policy grants public access, Block Public Access will block it
- A best practice: enable **account-level Block Public Access** for all buckets that should never be public

**Exam tip**: Account-level Block Public Access overrides bucket-level settings. If an account-level block is enabled, a bucket-level allow will NOT work.

### 1.4. S3 ACLs (Legacy — Not Recommended)

- Bucket ACLs: Grant cross-account access at the bucket level (pre-bucket policy mechanism)
- Object ACLs: Grant access at the individual object level
- ACLs are **legacy** — AWS recommends using bucket policies and IAM policies instead
- ACLs are disabled by default for new buckets (since April 2023); use **Object Ownership** to disable ACLs
- ACLs only grant **READ**, **WRITE**, **READ_ACP**, **WRITE_ACP**, **FULL_CONTROL** — they are less granular than policies

**Exam tip**: If a question mentions "legacy" or "object-level cross-account access," it may be testing ACLs. But the correct answer on the exam is usually bucket policies or presigned URLs instead of ACLs.

### 1.5. Object Ownership

- Controls who owns objects in the bucket and whether ACLs are disabled
- Three settings:

| Setting                     | Behavior                                                           |
| --------------------------- | ------------------------------------------------------------------ |
| **Bucket Owner Preferred**  | Objects written by other accounts become owned by the bucket owner |
| **Object Writer** (default) | The account that writes the object owns it                         |
| **Bucket Owner Enforced**   | Disables ACLs; bucket owner owns all objects automatically         |

- **Bucket Owner Enforced** is the recommended setting — it disables ACLs entirely and ensures the bucket owner always has full control

## 2. S3 Encryption

### 2.1. Encryption Types

| Type                       | Key Management              | Description                                                           |
| -------------------------- | --------------------------- | --------------------------------------------------------------------- |
| **SSE-S3**                 | AWS manages S3-managed keys | AES-256, automatic, free, transparent encryption at rest              |
| **SSE-KMS**                | AWS KMS                     | Envelope encryption with KMS, separate permissions, audit trail       |
| **SSE-C**                  | Customer-provided key       | You provide the encryption key in every request; S3 does not store it |
| **DSSE-KMS**               | AWS KMS (dual-layer)        | Two layers of encryption (S3 + KMS) with dual-layer protection        |
| **Client-side encryption** | Customer-managed (client)   | Encrypt data before uploading; S3 never sees plaintext                |

**Exam tip**: The exam frequently tests which encryption type to use based on requirements. Key differentiators:

- **SSE-S3**: Simplest, free, but no key rotation control or audit trail
- **SSE-KMS**: Separate permissions (envelope key), CloudTrail audit of key usage, key rotation control, but has KMS request costs and 5500-50000 req/s limit per key
- **SSE-C**: When you must manage the encryption keys yourself and cannot trust AWS to store them
- **DSSE-KMS**: Compliance requirements for dual-layer encryption; each object is encrypted twice
- **Client-side**: When data must be encrypted before leaving your environment (e.g., regulatory requirements)

### 2.2. SSE-KMS Details

- Uses KMS CMK (customer managed or AWS managed)
- S3 uses the `kms:EncryptionContext` of `s3.amazonaws.com` for audit
- Requires KMS permissions: `kms:Encrypt`, `kms:Decrypt`, `kms:GenerateDataKey`
- S3 creates a grant on the KMS key when you configure SSE-KMS
- CloudTrail logs each KMS `Decrypt` call — useful for detection
- **S3 Bucket Key** (see below) reduces KMS API calls from one per object to one per hour per bucket

**Exam scenario**: An application writing millions of small objects to S3 with SSE-KMS is getting throttled → enable **S3 Bucket Key** to reduce KMS API call volume.

### 2.3. S3 Bucket Key

- Reduces KMS request costs by using a **time-limited bucket-level key** (generated from the KMS key)
- Without Bucket Key: each PUT creates a KMS `GenerateDataKey` call
- With Bucket Key: S3 creates a bucket-level key (valid ~1 hour) and uses it for all objects in that period
- Results in up to **99% reduction** in KMS API calls
- Can be enabled at the bucket level when SSE-KMS is configured

### 2.4. Default Encryption

- Can configure a bucket to **automatically encrypt** all objects at rest
- Two settings: SSE-S3 (AES-256) or SSE-KMS
- If a PUT request specifies no encryption header, S3 applies the default encryption
- If a PUT request specifies an encryption header, the request header takes precedence
- Bucket Policy can _enforce_ a specific encryption type (using `s3:x-amz-server-side-encryption` condition key)

**Exam scenario**: You need to enforce SSE-KMS for all objects in a bucket → use a **bucket policy** with a Deny for `s3:PutObject` where `s3:x-amz-server-side-encryption` does not equal `aws:kms`. (Or separately, you could configure default encryption + bucket policy to enforce.)

### 2.5. Client-Side Encryption

| Method                         | Description                                                                |
| ------------------------------ | -------------------------------------------------------------------------- |
| **AWS KMS (GenerateDataKey)**  | Use KMS to generate a data key, encrypt client-side, upload encrypted data |
| **S3 Encryption Client (SDK)** | Client library that handles envelope encryption automatically              |
| **Third-party encryption**     | PGP, OpenSSL, or custom encryption before upload                           |

- Server does not see plaintext data
- Data is decrypted client-side after download
- Key management is entirely the customer's responsibility

## 3. S3 Versioning

- Once enabled, S3 keeps **multiple versions** of every object
- Protects against accidental deletion and overwrites
- **States**: Unversioned (default), Versioning-enabled, Versioning-suspended
- **Cannot be disabled** once enabled — can only be suspended
- Versioning is a prerequisite for: **Object Lock**, **MFA Delete**, **Replication**
- Deleted objects in versioned buckets create **delete markers** instead of permanent deletion
- To permanently delete: specify the version ID

**Exam scenario**: A user accidentally deleted an object and needs to recover it → if versioning is enabled, the object still exists as a prior version; remove the delete marker. If versioning is NOT enabled, the object is permanently deleted.

## 4. MFA Delete

- Requires **multi-factor authentication** to delete objects or suspend versioning
- Must be enabled on a **versioned bucket**; can only be enabled by the bucket owner (root account)
- Uses the `x-amz-mfa` header in the request with the MFA device serial number and code
- Once enabled, any `DeleteObject` or `PUT Bucket versioning` (to suspend) requires MFA
- Prevents accidental or malicious data deletion

**Exam scenario**: A compliance requirement demands that objects cannot be permanently deleted without a secondary authentication factor → enable **MFA Delete** on the versioned bucket.

## 5. S3 Object Lock (WORM)

- Write-Once-Read-Many (WORM) protection — objects cannot be overwritten or deleted for a specified retention period
- Requires **versioning** to be enabled
- **Two retention modes**:

| Mode           | Effect                                                                     | Can Be Shortened?        |
| -------------- | -------------------------------------------------------------------------- | ------------------------ |
| **Governance** | Users with special permissions (`s3:BypassGovernanceRetention`) can delete | No (but can be bypassed) |
| **Compliance** | No one can delete, including the root user; retention period is absolute   | No                       |

- **Legal Hold**: Prevents deletion of an object version indefinitely; no retention period; can be added/removed by users with `s3:PutObjectLegalHold` permission; no expiry date
- A retention period of **0 days** is valid (lock exists for 0 seconds, after which the protection ends)
- Default retention: Sets a default retention period (days/years) applied to all objects
- **Retention period is calculated from the object creation time**, not from the time the retention was applied

**Exam scenario**: An audit requirement mandates that log files are retained for 7 years and cannot be deleted or modified by anyone, including AWS administrators → use **S3 Object Lock** with **Compliance mode** and a 7-year retention period.

**Exam scenario**: A legal investigation requires that specific documents cannot be deleted until the investigation ends → use **S3 Object Lock Legal Hold** on the specific object versions.

## 6. Glacier Vault Lock

- Applies WORM controls to **S3 Glacier** vaults (not standard S3 buckets)
- Uses **Vault Lock policies** (IAM resource-based policies for Glacier vaults)
- The lock goes through a two-step process:
  1. **Initiate Vault Lock**: Attaches the policy to the vault
  2. **Complete Vault Lock**: Within 24 hours, you must complete the lock; wait or the policy is removed
- Once locked, the policy **cannot be changed**
- Allows compliance controls for archive data

**Exam tip**: Glacier Vault Lock is similar to S3 Object Lock but for Glacier vaults specifically. It uses a policy-based approach rather than per-object retention.

## 7. S3 Pre-signed URLs

- URLs that grant **temporary access** to S3 objects to anyone who has the URL
- Generated using the bucket owner's **AWS credentials** (IAM user or role)
- The credential used to generate the URL determines the permissions — the URL holder gets whatever permissions that credential has
- Default expiry: 1 hour (max: 7 days, or 12 hours with `AWSSignatureV2`)
- Can be used for: GET (download), PUT (upload), DELETE, and other operations
- **Signature v4** is the default and handles bucket keys and presigned URL expiry

**Exam scenario**: An application needs to allow users to upload files directly to S3 without AWS credentials → generate **pre-signed PUT URLs** for the upload.

**Exam scenario**: A user needs to share a large file from S3 for 24 hours without making the bucket public → generate a **pre-signed GET URL** with a 24-hour expiry.

| Use Case                     | Presigned URL Method                     |
| ---------------------------- | ---------------------------------------- |
| Download an object           | GET presigned URL                        |
| Allow upload to S3           | PUT presigned URL                        |
| Allow specific operations    | POST presigned URL                       |
| Allow conditional operations | Use presigned URL with condition headers |

**Exam scenario**: A pre-signed URL generated by User A is shared with User B. User B tries to access the object and gets AccessDenied → User A's permissions at the time of URL generation may have been revoked, the URL may have expired, or User A's credentials may have changed (e.g., key rotation). Pre-signed URLs are tied to the **original signer's permissions**.

## 8. S3 Access Points

- **Network endpoints** for S3 buckets that simplify managing access for large-scale data access
- Each access point has its own:
  - **Bucket policy** (separate from the main bucket policy)
  - **Network access control** (VPC origin or internet)
  - **Block Public Access** settings
- Access point ARN format: `arn:aws:s3:<region>:<account>:accesspoint/<ap-name>`
- Each access point can have its own **VPC origin** — only traffic from that VPC can access the bucket via the access point
- Access points are **not supported by all AWS services** — some services need direct bucket access
- Up to **1000 access points per account** per region

**Exam scenario**: Multiple applications need different access policies for the same S3 bucket — e.g., App1 needs read-only, App2 needs write-only, App3 needs VPC-restricted access → create **S3 Access Points** with distinct policies for each application.

**Exam scenario**: An S3 bucket should only be accessible from within a specific VPC, without using a public endpoint → create an **S3 Access Point** with **VPC origin** pointing to that VPC. Also add a **VPC Endpoint** (Gateway or Interface) for S3.

### 8.1. Access Point Policy vs Bucket Policy

- The access point policy is evaluated **in addition** to the bucket policy and IAM policy
- Both the **bucket policy** AND the **access point policy** must allow the action for it to succeed
- Access point policies can be more permissive than the bucket policy — but the bucket policy's denies still apply
- The access point policy attaches to the access point, not the bucket

### 8.2. Multi-Region Access Points

- Provide a single **global endpoint** that routes traffic to the nearest bucket replica across AWS Regions
- Used with S3 **Cross-Region Replication (CRR)**
- Automatic failover: if the primary region goes down, traffic is rerouted to a replica region
- Provides a single Access Point ARN that works globally
- Supports **failover control** (you can control which region is active)

**Exam scenario**: A global application needs a single endpoint to access S3 data with automatic failover across regions → use **S3 Multi-Region Access Points**.

## 9. S3 Object Lambda Access Points

- Allows you to run **Lambda functions** on data as it is retrieved from S3
- Transform data on-the-fly: redact PII, resize images, convert formats, etc.
- The application accesses data through the Object Lambda Access Point
- The Lambda function reads the original object from S3, transforms it, and returns the result
- No need to store transformed copies in S3

**Exam scenario**: An application needs to retrieve objects from S3 but must redact personally identifiable information (PII) before delivering them to certain users → use **S3 Object Lambda Access Points** with a Lambda function that redacts PII on-the-fly.

## 10. S3 Replication

### 10.1. Replication Types

| Type                               | Description                                              |
| ---------------------------------- | -------------------------------------------------------- |
| **Cross-Region Replication (CRR)** | Replicates objects to a bucket in a different AWS Region |
| **Same-Region Replication (SRR)**  | Replicates objects to a bucket in the same AWS Region    |

### 10.2. Key Features

- Requires **versioning** on both source and destination buckets
- Can replicate: objects, delete markers (optional), metadata changes
- Can replicate with **encryption**: SSE-S3, SSE-KMS (must configure KMS key permissions)
- Can replicate objects from **multiple source buckets** to a single destination bucket
- **Replication Time Control (RTC)**: Guarantees replication within 15 minutes (most objects replicate in seconds)
- **Replication metrics** and **S3 Events** are available for monitoring
- **IAM role**: S3 assumes a replication IAM role to perform the replication
- Replication requires the IAM role to have: `s3:GetObjectVersionForReplication`, `s3:ReplicateObject`, `s3:ReplicateDelete`, `s3:GetReplicationConfiguration`

### 10.3. What is NOT replicated by default

- Objects before replication was enabled (must use S3 Batch Replication)
- Objects in the source bucket that failed to replicate
- Delete markers (unless explicitly configured)
- Objects stored with **SSE-C** encryption
- Objects stored with **SSE-KMS** (requires additional KMS key permissions)
- **Object Lock** retention periods (replicated objects get the destination bucket's default retention)

### 10.4. Replication Use Cases

| Use Case                           | Replication Type | Notes                                     |
| ---------------------------------- | ---------------- | ----------------------------------------- |
| Disaster recovery (geo-redundancy) | CRR              | Store data in a secondary region          |
| Data sovereignty / compliance      | CRR              | Keep data in specific regions             |
| Reduce latency (hot data)          | SRR              | Replicate to bucket in same region        |
| Aggregate logs across accounts     | CRR/SRR          | Replicate to a centralized logging bucket |

**Exam scenario**: A company needs to comply with a regulation requiring automatic data replication to a secondary AWS Region for disaster recovery → enable **Cross-Region Replication (CRR)** between the source and destination buckets.

## 11. S3 Lifecycle Policies

- Automate transitioning objects between storage classes or deleting them
- Rules can apply to **current versions**, **previous versions**, or **delete markers**

| Transition                 | Min Days |
| -------------------------- | -------- |
| S3 Standard → IA           | 30       |
| S3 Standard → Glacier IR   | 30       |
| S3 Standard → Glacier      | 90       |
| S3 Standard → Deep Archive | 180      |
| S3 IA → Glacier            | 30       |
| S3 IA → Deep Archive       | 180      |

- **Expiration**: Permanently delete objects after a specified number of days
- **Incomplete multipart upload**: Abort after a specified number of days
- Lifecycle policies are evaluated **once per day**
- Lifecycle transitions are irreversible (cannot move from Glacier to Standard)

**Exam scenario**: Cost optimization — automatically transition logs from S3 Standard to S3 Glacier after 90 days and delete after 7 years → configure a **lifecycle policy** with transition and expiration rules.

## 12. S3 Event Notifications

- Send notifications when specific events occur in an S3 bucket
- Event types: `s3:ObjectCreated:*`, `s3:ObjectRemoved:*`, `s3:ObjectRestore:*`, `s3:Replication:*`
- Destinations: **SNS Topic**, **SQS Queue**, **Lambda Function**
- Can be filtered by **prefix** or **suffix**
- Event notifications are **not guaranteed** (at-least-once delivery, but duplicates and missed events are possible)

**Exam tip**: S3 events are typically delivered within seconds but can be delayed. For guaranteed event delivery, use **S3 Event Notifications with SQS** (durable queue) or **S3 Event Notifications to Lambda** (retry logic built into Lambda). S3 events send the bucket name, object key, size, and ETag.

## 13. Logging and Monitoring

### 13.1. S3 Server Access Logs

- Logs all requests made to a bucket (including denied requests)
- Logs are delivered to a **destination bucket** in the same AWS Region
- Log format includes: requester, bucket name, key, action, HTTP status, error code, bytes sent/received, etc.
- Logs are delivered on a **best-effort basis** — not guaranteed
- Logs can be up to several hours old
- Enable on the source bucket, specify destination bucket (must be in the same region)
- The destination bucket is **at least one hour delayed** from the actual request

**Exam scenario**: An organization needs a detailed record of ALL access to an S3 bucket, including denied requests, for forensic analysis → enable **S3 Server Access Logging** to a destination bucket in the same region.

### 13.2. AWS CloudTrail for S3

- **Management events**: Captured by default in all CloudTrail trails (e.g., `CreateBucket`, `PutBucketPolicy`, `PutBucketEncryption`)
- **Data events**: Must be explicitly enabled (e.g., `GetObject`, `PutObject`, `DeleteObject`). Additional cost.
- Data events can be filtered by **read** vs **write** events
- CloudTrail delivers logs to S3 (typically within 15 minutes of the API call)

**Exam scenario**: An organization needs near-real-time monitoring of S3 object-level API calls for security analysis → enable **CloudTrail data events** for S3 (read and write events), with CloudWatch Logs integration.

### 13.3. S3 vs Server Access Logs vs CloudTrail

| Feature             | S3 Server Access Logs                         | CloudTrail Data Events                     |
| ------------------- | --------------------------------------------- | ------------------------------------------ |
| **Granularity**     | Each request (GET, PUT, DELETE, HEAD, etc.)   | Each API call (GetObject, PutObject, etc.) |
| **Delivery time**   | Best-effort (hours delay)                     | Within 15 minutes                          |
| **Denied requests** | Included                                      | Included                                   |
| **Cost**            | Free (S3 storage costs apply)                 | Additional cost for data events            |
| **Reliability**     | Best-effort (may miss some)                   | Guaranteed delivery                        |
| **Configuration**   | Enable on bucket + specify destination bucket | Enable data events in CloudTrail trail     |

**Exam scenario**: You need guaranteed, near-real-time logging of all object access → use **CloudTrail** (not S3 Server Access Logs which are best-effort and delayed).

### 13.4. S3 Inventory

- Generates a **CSV/Parquet/ORC** report of all objects in a bucket and their metadata
- Reports are generated **daily** or **weekly**
- Can include: size, storage class, encryption status, replication status, last modified date, etag, object lock retention, multipart upload status
- Provides a **complete object listing** — useful for compliance, audit, and lifecycle management
- Cost: based on number of objects listed and destination storage

**Exam scenario**: You need to report on all objects stored in S3 that are encrypted with SSE-C for compliance → use **S3 Inventory** with an optional field for encryption status.

### 13.5. S3 Storage Lens

- Provides a **single dashboard** of storage usage and activity metrics across an entire AWS organization
- Metrics include: storage bytes, object count, bucket count, encryption status, replication success/failure, etc.
- **Default dashboard** is free; **advanced metrics** have additional cost
- Can generate **weekly/monthly reports** to a destination bucket
- Aggregates data at the organization, account, region, bucket, or prefix level

**Exam scenario**: A security team needs a centralized view of S3 storage metrics across all accounts in an AWS Organization, including encryption status → use **S3 Storage Lens** with advanced metrics.

## 14. Network Security for S3

### 14.1. VPC Endpoints for S3

| Type                   | Description                                                     | Uses                                                                                                    |
| ---------------------- | --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| **Gateway Endpoint**   | A target in the route table that routes traffic to S3 privately | **Most common** — free, uses Prefix List                                                                |
| **Interface Endpoint** | An ENI in the VPC with a private IP (AWS PrivateLink)           | Can be accessed from on-premises via VPN/Direct Connect, supports bucket policies with `aws:SourceVpce` |

**Gateway Endpoint**:

- Added to the VPC route table as a prefix list for S3
- Traffic stays within the AWS network (never goes to the internet)
- **Free** to use (no hourly charge)
- Accessible via bucket policies with `aws:SourceVpc` condition
- Cannot be used from on-premises (VPN/Direct Connect)

**Interface Endpoint**:

- Uses AWS PrivateLink — an elastic network interface in the subnet
- Accessible from on-premises via VPN/Direct Connect
- Supports bucket policies with `aws:SourceVpce` condition (specific VPC Endpoint ID)
- **Not free** — hourly charge + data processing fee
- Can be used from a different region (Inter-Region VPC Endpoint with PrivateLink)

**Exam scenario**: A private subnet in a VPC needs to access S3 without going through the internet → create a **Gateway VPC Endpoint** for S3, add it to the route table, and optionally use a bucket policy restricting access to the VPC.

### 14.2. VPC Origin for Access Points

- An S3 Access Point can be configured to only accept traffic from a specific VPC
- Works with both Gateway and Interface endpoints
- When enabled, all requests must come from the specified VPC
- The access point policy can further restrict access (e.g., by IAM role, user, or IP range)

### 14.3. Condition Keys for Network Restrictions

| Condition Key         | Use Case                                                    |
| --------------------- | ----------------------------------------------------------- |
| `aws:SourceIp`        | Restrict to specific IP address ranges                      |
| `aws:SourceVpc`       | Restrict to requests from a specific VPC (Gateway Endpoint) |
| `aws:SourceVpce`      | Restrict to requests from a specific VPC Endpoint ID        |
| `aws:SecureTransport` | Require HTTPS (deny HTTP)                                   |

**Exam scenario**: Enforce encryption in transit → use bucket policy with `aws:SecureTransport: false` → Deny. Combined with default encryption (at rest), this provides full in-transit + at-rest encryption.

## 15. S3 Condition Keys (Important for Exam)

| Condition Key                                    | Description                                                |
| ------------------------------------------------ | ---------------------------------------------------------- |
| `s3:x-amz-server-side-encryption`                | Enforce SSE type (`AES256` or `aws:kms`)                   |
| `s3:x-amz-server-side-encryption-aws-kms-key-id` | Enforce a specific KMS key ID for SSE-KMS                  |
| `s3:x-amz-acl`                                   | Enforce specific ACL (legacy)                              |
| `s3:versionid`                                   | Control access to specific object versions                 |
| `s3:object-lock-mode`                            | Enforce Object Lock mode (`GOVERNANCE` or `COMPLIANCE`)    |
| `s3:object-lock-retain-until-date`               | Enforce a specific retention date                          |
| `s3:object-lock-remaining-retention-days`        | Enforce minimum remaining retention days                   |
| `s3:object-lock-legal-hold-status`               | Enforce legal hold status (`ON` or `OFF`)                  |
| `s3:signatureversion`                            | Require Signature Version 4 (deny v2 which is less secure) |
| `s3:signatureage`                                | Maximum age of the presigned URL signature                 |
| `s3:delimiter`                                   | Control access to list operations with a delimiter         |
| `s3:prefix`                                      | Control access to objects with a specific prefix           |
| `s3:locationconstraint`                          | Enforce specific region for bucket creation                |
| `s3:object-lambda:transformation-configuration`  | Control which Object Lambda transformations can be used    |

**Exam scenario**: You must enforce that only specific IAM roles can upload objects with a specific KMS key → use a bucket policy with `s3:x-amz-server-side-encryption-aws-kms-key-id` condition to require the specific KMS key ARN.

**Exam scenario**: A policy requires that objects uploaded to S3 must be encrypted at rest with AES-256 → use bucket policy with `s3:x-amz-server-side-encryption` condition key requiring `AES256`.

## 16. Cross-Account Access Patterns

### 16.1. Bucket Policy (Resource-Based)

```
Account A (Bucket Owner) ──bucket policy grant──→ Account B (External User)
```

- Bucket policy in Account A grants access to Account B's root user or IAM entities
- Account B must also have an IAM policy allowing the action
- No need to create an IAM role
- The external principal uses the **bucket's regional endpoint**

**Exam scenario**: Account A needs to give Account B access to its bucket → Account A creates a bucket policy granting `s3:GetObject` to Account B's root user, AND Account B creates an IAM policy allowing the same action.

### 16.2. IAM Role (Cross-Account)

```
Account A (Bucket Owner) ──IAM role trust──→ Account B (External User)
```

- Account A creates an IAM role with S3 permissions
- Account B's users assume the IAM role (using STS `AssumeRole`)
- The role trust policy allows Account B to assume the role
- More secure than a bucket policy — access is via role assumption, not direct

**Exam scenario**: Account A needs to give Account B access to its S3 bucket with the ability to audit exactly who accessed the data → create an IAM role in Account A, grant it S3 permissions, and configure the trust policy to allow Account B to assume the role.

### 16.3. Cross-Account Access — Which Method?

| Method            | When to Use                                                                                            |
| ----------------- | ------------------------------------------------------------------------------------------------------ |
| **Bucket Policy** | Simple, direct access; no need to track who accessed; Account B's users want direct access             |
| **IAM Role**      | Need fine-grained audit; accessing across many buckets; Account B's users need temporary credentials   |
| **Access Points** | Multiple applications with different policies on the same bucket; per-application network restrictions |

## 17. S3 Transfer Acceleration

- Uses AWS **edge locations** to accelerate uploads over long distances (using AWS backbone network)
- Provides a **distinct URL** for the bucket: `{bucket}.s3-accelerate.amazonaws.com`
- **Additional cost** based on accelerated bytes
- Only beneficial for **uploads** (not downloads) over long distances
- Must be enabled on the bucket

**Exam scenario**: Users in Europe are experiencing slow upload speeds to an S3 bucket in us-west-2 → enable **S3 Transfer Acceleration**.

## 18. S3 Requester Pays

- The **requester** (downloader) pays for the data transfer and request costs, not the bucket owner
- The requester must be an authenticated AWS account (anonymous access is not allowed)
- Used for: sharing large datasets, reducing costs for data providers
- Must be enabled on the bucket
- The requester must include `x-amz-request-payer` in the request header

**Exam scenario**: A research organization shares large public datasets and wants to avoid data transfer costs → enable **Requester Pays** on the bucket.

## 19. S3 CORS (Cross-Origin Resource Sharing)

- Controls how S3 buckets respond to cross-origin requests from web browsers
- Configured in the bucket's **CORS configuration** (XML or JSON)
- Example use case: A web app on `https://app.example.com` loads assets from `https://my-bucket.s3.amazonaws.com`
- CORS rules specify: AllowedOrigins, AllowedMethods, AllowedHeaders, ExposeHeaders, MaxAgeSeconds

**Exam scenario**: A web application hosted on a domain different from the S3 bucket's domain is unable to access objects in the browser → configure **CORS** on the S3 bucket.

## 20. S3 Batch Operations

- Perform bulk actions on a **large number of objects** (billions of objects)
- Actions include: copy, encrypt, restore from Glacier, set ACLs/tags, invoke Lambda function
- Uses an **S3 Inventory** report or a CSV manifest to specify objects
- Tracks progress and sends notifications on completion
- Can be used to **encrypt unencrypted objects** or **change encryption type** across all objects in a bucket

**Exam scenario**: An organization discovers thousands of existing S3 objects that are not encrypted → use **S3 Batch Operations** to apply SSE-S3 or SSE-KMS encryption to all objects.

## 21. S3 Select & Glacier Select

- Retrieve **subsets of data** from objects using SQL queries (server-side filtering)
- S3 Select works on CSV, JSON, Parquet, and BZIP2-compressed files
- Reduces data transfer costs and latency (less data transferred over the network)
- Supports simple SQL queries: `SELECT s._1 FROM s3object s LIMIT 100`
- Not a security feature per se, but relevant for cost-effective data access patterns

## 22. Security Best Practices for S3

| Practice                                                | Description                                                     |
| ------------------------------------------------------- | --------------------------------------------------------------- |
| **Enable Block Public Access**                          | Account-level — prevents accidental public exposure             |
| **Use bucket policies for access control**              | Granular, condition-based, supports cross-account               |
| **Disable ACLs (use Object Ownership Enforced)**        | Legacy; use policies instead                                    |
| **Enable default encryption**                           | Ensures all objects are encrypted at rest                       |
| **Enforce HTTPS**                                       | Bucket policy with `aws:SecureTransport`                        |
| **Enable versioning**                                   | Protects against accidental deletion and overwrites             |
| **Enable MFA Delete**                                   | Critical for governance, risk, compliance scenarios             |
| **Use Object Lock for WORM compliance**                 | Example: financial records, audit logs                          |
| **Use presigned URLs**                                  | Grant temporary, scoped access without making data public       |
| **Enable server access logs or CloudTrail data events** | Monitor access patterns and detect anomalies                    |
| **Use S3 Inventory**                                    | Track encryption status, replication status, object metadata    |
| **Configure VPC Endpoints**                             | Keep S3 traffic within the AWS network                          |
| **Use S3 Access Points**                                | Simplify multi-application policy management                    |
| **Use S3 Bucket Key**                                   | Reduce KMS costs when using SSE-KMS                             |
| **Apply least privilege**                               | Grant minimal S3 permissions needed per principal               |
| **Use IAM roles**                                       | Avoid long-term access keys for S3 access                       |
| **Use S3 Object Lambda**                                | Redact sensitive data on-the-fly at access time                 |
| **Monitor S3 with Macie**                               | Detect and protect sensitive data (PII, financial, credentials) |

## 23. Amazon Macie Integration

- **Amazon Macie** uses machine learning to discover, classify, and protect sensitive data in S3
- Automatically generates an **inventory of S3 buckets** and evaluates them for security and access control
- Identifies **publicly accessible buckets**, **unencrypted buckets**, and **buckets shared with AWS accounts outside the organization**
- Creates **findings** when sensitive data (PII, credentials, financial data) is detected
- Integrates with **AWS Security Hub**, **EventBridge**, and **Amazon Detective**
- Macie only analyzes **S3** data (not EBS, RDS, or other services)

**Exam scenario**: A security team needs to automatically detect PII in S3 buckets and get notified when it is found → deploy **Amazon Macie** with automated discovery jobs and integrate with EventBridge/SNS for notifications.

## 24. S3 Service Control Policies (SCPs)

- Organizations SCPs can restrict S3 actions across entire accounts or OUs
- Examples:
  - Deny `s3:PutBucketPolicy` unless the policy restricts public access
  - Deny `s3:PutBucketPublicAccessBlock` to ensure Block Public Access cannot be modified
  - Deny `s3:PutEncryptionConfiguration` to enforce specific encryption settings

**Exam scenario**: An organization wants to prevent any account from making S3 buckets publicly accessible → create an SCP that denies `s3:PutBucketPolicy` where the policy would grant public access (using `s3:PolicyHasPublicAccess` or similar condition check).

## 25. S3 Storage Classes (Security-Relevant)

| Storage Class                     | Min Duration | Retrieval Time   | Use Case                                  |
| --------------------------------- | ------------ | ---------------- | ----------------------------------------- |
| **S3 Standard**                   | None         | Immediate        | Frequently accessed data                  |
| **S3 Intelligent-Tiering**        | None         | Immediate        | Unknown or changing access patterns       |
| **S3 Standard-IA**                | 30 days      | Immediate        | Infrequently accessed but rapid retrieval |
| **S3 One Zone-IA**                | 30 days      | Immediate        | Non-critical, recreatable data            |
| **S3 Glacier Instant Retrieval**  | 90 days      | Milliseconds     | Archive data needing instant access       |
| **S3 Glacier Flexible Retrieval** | 90 days      | Minutes to hours | Archive data, long-term backup            |
| **S3 Glacier Deep Archive**       | 180 days     | 12-48 hours      | Long-term archival, compliance            |
| **S3 Outposts**                   | None         | Immediate        | On-premises S3 (hybrid cloud)             |

**Security note**: One Zone-IA stores data in a single Availability Zone — if the AZ fails, data is lost. For security-sensitive workloads, use Standard, Standard-IA, or Glacier classes.

## 26. S3 Transfer Costs (Security-Relevant Cost Considerations)

| Direction                    | Cost                            |
| ---------------------------- | ------------------------------- |
| Upload to S3                 | Free                            |
| Download from S3 to internet | Charged (varies by region)      |
| S3 to CloudFront             | Free (reduced data egress cost) |
| S3 to EC2 (same region)      | Free                            |
| S3 to EC2 (different region) | Charged                         |

**Security note**: CloudFront can act as a security layer in front of S3 — you can use **Origin Access Identity (OAI)** or **Origin Access Control (OAC)** to ensure only CloudFront can access the S3 bucket, and CloudFront provides DDoS protection (AWS Shield).

## 27. S3 and CloudFront (Origin Access Control)

- **Origin Access Identity (OAI)**: Legacy method — a special CloudFront identity that authenticates to S3
- **Origin Access Control (OAC)**: Newer, recommended method — supports:
  - **SigV4** signing for requests
  - **Enforce HTTPS** between CloudFront and S3
  - **Cross-region buckets**
  - **SSE-KMS** encrypted buckets
- With OAI/OAC, you configure the bucket policy to only allow access from the CloudFront distribution

**Exam scenario**: You need to serve content from S3 through CloudFront and ensure users can only access S3 via CloudFront, not directly → use **Origin Access Control (OAC)** with a bucket policy that denies any access not from the CloudFront distribution.

## 28. Audit and Compliance Patterns

| Requirement                      | Solution                                                       |
| -------------------------------- | -------------------------------------------------------------- |
| Log all access to S3 (detailed)  | S3 Server Access Logs + CloudTrail data events                 |
| Alert on public bucket creation  | AWS Config managed rule `s3-bucket-public-read-prohibited`     |
| Detect sensitive data in S3      | Amazon Macie                                                   |
| Monitor S3 configuration changes | AWS Config rules for S3                                        |
| Enforce encryption at rest       | Default encryption + bucket policy condition                   |
| Enforce encryption in transit    | Bucket policy with `aws:SecureTransport` Deny                  |
| Prevent accidental deletion      | Object Lock + MFA Delete + versioning                          |
| Prove WORM compliance            | Object Lock (Compliance mode) or Glacier Vault Lock            |
| Centralized S3 visibility        | CloudTrail Organization Trail + S3 Storage Lens + Security Hub |

## 29. Common Exam Scenarios

1. **Public bucket exposure**: A bucket is accidentally made public. Exam asks how to prevent this → enable **Block Public Access** at the account level.

2. **Enforce encryption**: A security policy requires all S3 objects to be encrypted at rest → enable **default encryption** on the bucket AND use a **bucket policy** with `s3:x-amz-server-side-encryption` condition to enforce encryption on PutObject (catch objects being PUT without encryption headers).

3. **Cross-account access**: Account B needs to read objects from Account A's bucket → **bucket policy** in Account A granting `s3:GetObject` to Account B + **IAM policy** in Account B allowing `s3:GetObject`.

4. **Pre-signed URL fails**: A pre-signed URL expires after the configured timeout, or the signing user's permissions are revoked → generate a new URL with sufficient expiry or ensure the signing user has ongoing permissions.

5. **KMS throttling on S3**: Writing many small objects to SSE-KMS encrypted S3 bucket → enable **S3 Bucket Key** to reduce KMS calls.

6. **VPC-only bucket**: An S3 bucket must only be accessible from a specific VPC → **Gateway Endpoint** + bucket policy with `aws:SourceVpc` condition, or **Access Point with VPC origin**.

7. **Object recovery**: A user deleted an object → if versioning is enabled, remove the delete marker. If not, the object is **permanently lost**.

8. **Compliance WORM**: Data must be immutable for 7 years → **Object Lock** in **Compliance mode** with a 7-year retention period.

9. **Sensitive data redaction**: An application needs to mask PII in S3 objects for certain users → **S3 Object Lambda Access Point** with a Lambda transformation function.

10. **Logging for investigation**: Need detailed forensic logs of all S3 access, including denied requests → **S3 Server Access Logs** (free, detailed) + **CloudTrail data events** (near-real-time, guaranteed).

## 30. Exam Tips

1. **Block Public Access** overrides everything — even if a bucket policy explicitly allows public access.

2. **Bucket policy + IAM policy**: For cross-account access via bucket policy, the external account still needs an IAM policy allowing the S3 action.

3. **SSE-S3**: Free, automatic, transparent. **SSE-KMS**: Separate permissions, audit trail, but has cost and throttling limits. **SSE-C**: You manage keys, S3 does not store them. **DSSE-KMS**: Dual-layer.

4. **Default encryption** applies only to objects uploaded WITHOUT encryption headers. If a PUT specifies encryption, the header takes precedence.

5. **S3 Bucket Key** reduces KMS API call volume by up to 99% — answer when SSE-KMS is causing throttling or high costs.

6. **Versioning** is a prerequisite for Object Lock, MFA Delete, and Replication.

7. **MFA Delete** requires versioning and can only be enabled by the root user (or users with the appropriate permissions). It prevents permanent deletion without MFA.

8. **Object Lock Compliance mode** is absolute — even the root user cannot delete objects. Use with extreme caution.

9. **Glacier Vault Lock** is for Glacier vaults, **Object Lock** is for S3 buckets. Both provide WORM, but Glacier Vault Lock uses a policy-based approach.

10. **Pre-signed URLs** expire after the configured time (max 7 days). If the signing user's permissions change after generation, the URL may fail.

11. **S3 Access Points** simplify policy management for multiple applications sharing a bucket.

12. **Multi-Region Access Points** provide a global endpoint with automatic failover — combine with CRR.

13. **S3 Object Lambda Access Points** transform data on-the-fly — useful for PII redaction, image resizing, format conversion.

14. **CRR vs SRR**: CRR for cross-region (DR, compliance). SRR for same-region (log aggregation, prod/test sync).

15. **Server Access Logs** are best-effort and delayed. **CloudTrail data events** are guaranteed and near-real-time.

16. **SCPs** can prevent public bucket creation, encryption modification, and other security-sensitive S3 changes across an organization.

17. **Macie** detects sensitive data in S3. It is the answer when a question involves finding PII, credentials, or financial data in S3.

18. **AWS Config rules** for S3: `s3-bucket-public-read-prohibited`, `s3-bucket-public-write-prohibited`, `s3-bucket-ssl-requests-only`, `s3-bucket-server-side-encryption-enabled`.

19. **S3 Storage Lens** for centralized storage metrics across accounts. **S3 Inventory** for detailed per-object reports.

20. **CloudFront + OAC** is the recommended way to serve S3 content through a CDN while blocking direct S3 access.
