# AWS CloudHSM

<figure>
  <img src="./images/Arch_AWS-CloudHSM_64@5x.png" alt="AWS CloudHSM Icon" width=200>
  <figcaption><center>AWS CloudHSM<br><i>Image source: AWS Documentation</i></center></figcaption>
</figure>

**Overview**: AWS CloudHSM provides dedicated hardware security modules (HSMs) in the AWS Cloud. It is a **customer-managed** service — you have full control over the HSM, including key management, users, and policies. CloudHSM is used when compliance requirements mandate that encryption keys be stored in FIPS 140-2 Level 3 validated hardware.

**Domain weight**: CloudHSM appears in the Data Protection domain (~18% of SCS-C03). It is typically the answer for scenarios requiring dedicated HSM control, specific cryptographic interfaces, or compliance mandates above what KMS provides.

## 1. What CloudHSM Provides

| Feature                    | Details                                                               |
| -------------------------- | --------------------------------------------------------------------- |
| **Dedicated HSM hardware** | Each HSM is a dedicated appliance in your AWS account                 |
| **FIPS 140-2 Level 3**     | Physical tamper-evident HSM — validated at Level 3                    |
| **Customer-managed**       | You manage users, keys, and policies — AWS has no access to your keys |
| **Standard APIs**          | PKCS#11, Java Cryptography Extensions (JCE), Microsoft CNG/CSP        |
| **Multi-AZ cluster**       | HSM cluster across AZs for high availability                          |
| **VPC-based**              | Deployed in your VPC with private IP addresses                        |

## 2. CloudHSM vs KMS

| Feature                     | CloudHSM                           | KMS                                                  |
| --------------------------- | ---------------------------------- | ---------------------------------------------------- |
| **Management**              | Customer-managed HSM cluster       | AWS-managed service                                  |
| **Key access**              | Only you have access to keys       | AWS has access to keys (unless custom key store)     |
| **FIPS level**              | FIPS 140-2 Level 3                 | FIPS 140-2 Level 2 (or Level 3 via custom key store) |
| **API interface**           | PKCS#11, JCE, Microsoft CNG/CSP    | KMS API (Encrypt, Decrypt, etc.)                     |
| **AWS service integration** | No direct integration              | Integrated with 70+ AWS services                     |
| **Scalability**             | Manual (provision additional HSMs) | Automatic                                            |
| **Cost**                    | Per HSM per hour                   | $1/month per key + usage                             |
| **Multi-tenant**            | No (dedicated HSM per customer)    | Yes (shared infrastructure)                          |

**Exam scenario**: A financial application needs to use PKCS#11 interface for cryptographic operations → **CloudHSM** is required. KMS does not support PKCS#11.

**Exam scenario**: A compliance requirement mandates FIPS 140-2 Level 3 for all cryptographic operations → **CloudHSM** (or KMS custom key store backed by CloudHSM) is required.

## 3. KMS Custom Key Store (CloudHSM Integration)

- KMS can create a **custom key store** backed by CloudHSM
- KMS keys in a custom key store are stored in your CloudHSM cluster
- Benefits: KMS's easy API (Encrypt, Decrypt, etc.) + CloudHSM's dedicated HSM
- Integrates with AWS services through KMS, but the keys are in your CloudHSM
- If the CloudHSM cluster is unavailable, the keys become unavailable (all operations fail)

**Exam scenario**: An application needs the ease of KMS API integration with AWS services but also requires FIPS 140-2 Level 3 key storage → use **KMS custom key store** backed by CloudHSM.

## 4. Cluster Architecture

### 4.1. High Availability

- Deploy at least **2 HSMs in different AZs** for high availability
- HSM cluster can scale to 28 HSMs
- Data is synchronized across all HSMs in the cluster automatically

### 4.2. Network

- HSMs are deployed in your VPC with **private IP addresses**
- Access via **security group rules** — EC2 clients must be allowed to connect
- No public endpoint — all access is within the VPC
- For cross-VPC access, use VPC peering or Transit Gateway

### 4.3. Client Access

- Applications connect to the HSM cluster using the **CloudHSM client software** on EC2
- Client uses PKCS#11, JCE, or CNG/CSP libraries to communicate with the HSM
- The client communicates with all HSMs in the cluster for load balancing and failover

## 5. User and Key Management

### 5.1. User Types

| User Type                   | Permissions                                             |
| --------------------------- | ------------------------------------------------------- |
| **PCO (Precrypto Officer)** | Create and delete crypto officers — privileged user     |
| **CO (Crypto Officer)**     | Create, delete, manage keys and users                   |
| **CU (Crypto User)**        | Use keys for cryptographic operations (encrypt/decrypt) |
| **AU (Appliance User)**     | Backup and restore HSM data                             |

- You manage users and keys directly on the HSM — AWS cannot access your keys
- Users are authenticated via passwords or mutual TLS

### 5.2. Key Management

- Full control over key generation, import, export, and deletion
- Keys never leave the HSM unencrypted (unless exported with permission)
- Key backup to S3 (encrypted by the HSM)
- Supported key types: AES, RSA, ECC, DES, HMAC, and others

## 6. Backup and Restore

- HSM data can be backed up to an S3 bucket in the same region
- Backups are encrypted by the HSM and can only be restored to a CloudHSM cluster
- Backups can be restored to a different cluster (for disaster recovery)
- A backup includes: all keys, all user identities (without passwords)

## 7. Use Cases

| Use Case                          | Why CloudHSM                                            |
| --------------------------------- | ------------------------------------------------------- |
| **FIPS 140-2 Level 3 compliance** | Only CloudHSM provides Level 3 validation               |
| **PKCS#11 interface requirement** | CloudHSM supports PKCS#11; KMS does not                 |
| **Full key ownership**            | AWS cannot access keys stored in CloudHSM               |
| **Oracle TDE / SQL Server TDE**   | Database Transparent Data Encryption with dedicated HSM |
| **SSL/TLS offloading**            | Store private keys in CloudHSM for HTTPS termination    |
| **Code signing**                  | Protect code signing private keys in FIPS-validated HSM |

**Exam scenario**: A company uses Oracle Transparent Data Encryption (TDE) and needs to store the encryption key in a FIPS-validated HSM → use **CloudHSM** (Oracle TDE supports PKCS#11 library).

## 8. Cost

| Cost Driver            | Rate                                        |
| ---------------------- | ------------------------------------------- |
| **HSM per hour**       | ~$1.50 per HSM per hour (varies by region)  |
| **Minimum deployment** | 2 HSMs (recommended for HA) = ~$2,000/month |
| **S3 backup storage**  | Standard S3 pricing                         |
| **Data transfer**      | Standard data transfer costs                |

- CloudHSM is significantly more expensive than KMS
- Always compare the compliance requirement vs cost before choosing CloudHSM

## 9. Limits

| Resource                    | Limit                      |
| --------------------------- | -------------------------- |
| HSMs per cluster            | 28                         |
| HSMs per account per region | 3 (soft limit)             |
| Backup storage              | Unlimited (S3)             |
| Users per HSM               | 1,024                      |
| Keys per HSM                | 3,000 (varies by key type) |

## 10. Security Best Practices

- **Deploy at least 2 HSMs** in different AZs — no single point of failure
- **Enable multi-factor authentication** for HSM users (password + client certificate)
- **Backup HSM data regularly** to S3
- **Use security groups** to restrict client access to the HSM
- **Rotate HSM user passwords** regularly
- **Use VPC endpoints** for backup operations (avoid internet)
- **Deploy client in private subnets** — no public access to the HSM

## 11. Exam Tips

1. **CloudHSM = dedicated HSM, customer-managed**. KMS = managed service, AWS-controlled.

2. **FIPS 140-2 Level 3** is the primary differentiator — KMS is Level 2 (or Level 3 with custom key store).

3. **PKCS#11, JCE, Microsoft CNG/CSP** — CloudHSM supports these standard cryptographic APIs. KMS does not.

4. **KMS custom key store** combines KMS API with CloudHSM backing — the closest integration between the two.

5. **AWS cannot access your keys** in CloudHSM. For most scenarios KMS is sufficient; CloudHSM is for specific compliance needs.

6. **Cluster deployment**: Multiple HSMs across AZs for HA. Client connects to the cluster via the CloudHSM client.

7. **User types**: PCO (admin), CO (key/user management), CU (key usage), AU (backup). Know the difference.

8. **Backup to S3** — encrypted by the HSM, can only be restored to a CloudHSM cluster.

9. **Oracle TDE**: CloudHSM supports Oracle and SQL Server TDE via PKCS#11.

10. **Cost**: ~$1.50/HSM/hour, minimum 2 HSMs recommended = ~$2,000/month. Much more expensive than KMS.

11. **CloudHSM has no direct AWS service integration** — EC2 applications use PKCS#11/JCE to communicate. AWS services (S3, RDS) cannot directly use CloudHSM keys.

12. **VPC-based**: CloudHSM gets a private IP in your VPC. No public endpoint.

13. **If CloudHSM is unavailable** (e.g., HSM failure), all cryptographic operations fail. Multi-AZ HA is critical.

14. **KMS vs CloudHSM**: Use KMS unless you need FIPS 140-2 Level 3, PKCS#11, or complete key ownership.

15. **CloudHSM is not a substitute for KMS** — they serve different needs. KMS for integrated encryption, CloudHSM for dedicated HSM management.
