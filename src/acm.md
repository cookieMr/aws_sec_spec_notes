# AWS Certificate Manager (ACM) & Private CA

<figure>
  <img src="./images/Arch_AWS-Certificate-Manager_64@5x.png" alt="AWS Certificate Manager Icon" width=200>
  <figcaption><center>AWS Certificate Manager<br><i>Image source: AWS Documentation</i></center></figcaption>
</figure>

**Overview**: AWS Certificate Manager (ACM) handles the complexity of creating, storing, and renewing SSL/TLS certificates. ACM integrates with CloudFront, ALB, API Gateway, and other AWS services to provide encryption in transit. **ACM Private CA** extends this to issue private certificates for internal use.

**Domain weight**: ACM appears in the Data Protection domain (~18% of SCS-C03) and Infrastructure Security domain. It is the primary answer for encryption in transit (TLS/SSL) scenarios.

## 1. ACM Public Certificates

| Feature         | Details                                                        |
| --------------- | -------------------------------------------------------------- |
| **Cost**        | **Free** — public certificates issued by ACM are free          |
| **Renewal**     | Automatic (ACM renews 60 days before expiry)                   |
| **Validity**    | Up to 13 months                                                |
| **Integration** | CloudFront, ALB, API Gateway, Elastic Beanstalk, AppSync, etc. |
| **Validation**  | DNS validation or Email validation                             |

### 1.1. Supported Services

ACM public certificates can only be used with specific AWS services:

- **CloudFront** (must be in us-east-1 for CloudFront)
- **Application Load Balancer** (regional)
- **Network Load Balancer** (regional — via listener TLS termination)
- **API Gateway** (custom domain names)
- **Elastic Beanstalk** (via ALB)
- **AppSync**
- **Cognito Identity Pools**
- **AWS Nitro Enclaves**

**Exam scenario**: An application uses an ALB and needs HTTPS with automatic certificate renewal → create a certificate in **ACM** and associate it with the ALB listener — ACM handles renewal automatically.

### 1.2. Validation Methods

| Method    | How It Works                                | Pros                                    | Cons                                |
| --------- | ------------------------------------------- | --------------------------------------- | ----------------------------------- |
| **DNS**   | Add a CNAME record to the domain's DNS zone | No email needed, can auto-renew via DNS | Requires DNS zone access (Route 53) |
| **Email** | ACM sends email to domain admin addresses   | Works without DNS access                | Manual, cannot auto-renew           |

- **DNS validation is preferred** because ACM can automatically renew certificates without manual intervention
- ACM must be able to **re-validate the domain** before renewal — if DNS validation is used and the CNAME record is still present, ACM renews automatically
- For email validation, you must respond to the validation email each time

**Exam scenario**: An organization wants ACM certificates to automatically renew without manual intervention → use **DNS validation** (add the CNAME record to Route 53 once; ACM auto-renews).

### 1.3. Regions

- ACM certificates are **regional** — you must create a certificate in each region where you use it
- **Exception**: CloudFront requires certificates in **us-east-1** (the CloudFront global endpoint)
- If you need the same certificate in multiple regions, create a certificate in each region or use an imported certificate

## 2. ACM Private CA

### 2.1. Purpose

- Create and manage private certificates within your organization
- Issue certificates for internal-only use (no public trust)
- Full control over certificate validity periods, key algorithms, and revocation

### 2.2. Architecture

| Component                       | Description                                                                     |
| ------------------------------- | ------------------------------------------------------------------------------- |
| **Root CA**                     | The top-level CA — signs subordinate CAs (should be offline and secure)         |
| **Subordinate CA**              | Issues end-entity certificates (the CA that ACM uses for day-to-day operations) |
| **End-entity certificates**     | The actual TLS certificates used by applications                                |
| **Certificate revocation list** | List of revoked certificates (published to S3)                                  |

- ACM Private CA manages the CA hierarchy for you
- You can create: an entirely private CA hierarchy or a subordinate CA signed by an external root CA

### 2.3. Use Cases

- Internal TLS/SSL for internal ALBs, microservices, containers, IoT devices
- Client certificate authentication (mutual TLS / mTLS)
- Certificate-based authentication for VPNs, Wi-Fi, code signing
- Document signing

**Exam scenario**: A microservice architecture uses internal ALBs for service-to-service communication. The security team wants TLS encryption between services using internal certificates → use **ACM Private CA** to issue private certificates for the internal ALBs.

### 2.4. Private Certificates vs Public Certificates

| Feature        | Private Certificate                                       | Public Certificate                  |
| -------------- | --------------------------------------------------------- | ----------------------------------- |
| **Cost**       | ACM Private CA = paid ($400/month + per-certificate fees) | ACM public = free                   |
| **Trust**      | Trusted only within your organization                     | Trusted by all browsers and devices |
| **Validity**   | Configurable (1 day to 10 years)                          | Up to 13 months                     |
| **Renewal**    | Configurable                                              | Fully automatic                     |
| **Revocation** | You manage CRL                                            | ACM manages                         |

### 2.5. Private CA Pricing

- **CA creation**: $400/month per private CA (or $1/hour for each CA instance)
- **Certificates**: ~$0.75 per certificate per month (prorated)
- This is an important cost consideration — private certificates are not free

## 3. Imported Certificates

### 3.1. When to Use

- You have an existing certificate from a third-party CA (e.g., DigiCert, Let's Encrypt)
- You need a certificate in a region where ACM does not issue certificates for that domain
- You need a certificate for services that ACM does not integrate with (EC2 directly, NLB for end-to-end TLS)

### 3.2. Limitations

- **No automatic renewal** — you must manually re-import before expiry
- ACM sends **expiry notifications** via CloudWatch events 45, 30, 15, 7, and 3 days before expiry
- Must be PEM-encoded: certificate, private key, and optionally the certificate chain

**Exam scenario**: A company uses a third-party CA (e.g., DigiCert) for all public certificates and wants to use them with ALB → **import the certificate** into ACM. Note: renewal must be done manually.

## 4. Certificate Renewal

| Certificate Source                | Renewal Method                              |
| --------------------------------- | ------------------------------------------- |
| **ACM public (DNS validation)**   | Fully automatic (no action needed)          |
| **ACM public (Email validation)** | Manual (must respond to email each time)    |
| **ACM Private CA**                | Automatic or manual (configurable validity) |
| **Imported certificate**          | Manual (re-import before expiry)            |

- ACM sends CloudWatch events for certificate expiry:
  - 45 days before expiry
  - 30 days before expiry
  - 15 days before expiry
  - 7 days before expiry
  - 3 days before expiry

**Exam scenario**: An imported certificate is about to expire and needs to be updated → re-import the renewed certificate into ACM with the same certificate ARN. ACM will automatically update services using the certificate.

## 5. Certificate Types and Key Algorithms

| Algorithm           | ACM Public | ACM Private CA |
| ------------------- | ---------- | -------------- |
| **RSA 2048**        | Yes        | Yes            |
| **RSA 4096**        | Yes        | Yes            |
| **ECC 256 (P-256)** | Yes        | Yes            |
| **ECC 384 (P-384)** | Yes        | Yes            |

- ECC keys provide equivalent security with smaller key sizes (faster TLS handshake)
- RSA 2048 is the most widely compatible

## 6. Integration with Other Services

| Service                | Integration                                             |
| ---------------------- | ------------------------------------------------------- |
| **CloudFront**         | ACM certificate in us-east-1 for HTTPS viewer requests  |
| **ALB**                | ACM certificate for HTTPS listeners                     |
| **NLB**                | ACM certificate for TLS listeners                       |
| **API Gateway**        | Custom domain name with ACM certificate                 |
| **CloudFormation**     | Create ACM certificates and Private CAs via templates   |
| **Route 53**           | DNS validation via CNAME records                        |
| **IAM Roles Anywhere** | Use ACM Private CA for certificate-based authentication |

## 7. Security Best Practices

- **Use DNS validation** for automatic renewal
- **Use ACM public certificates** over imported certificates when possible (automatic renewal)
- **Use ACM Private CA** for internal TLS — do not use public certificates for internal services
- **Monitor certificate expiry** via CloudWatch events for imported certificates
- **Restrict certificate creation** via IAM policies (`acm:RequestCertificate`, `acm:ImportCertificate`)
- **Enable CloudTrail** to audit ACM API calls
- **Use short validity periods** for private certificates (improves security posture)
- **Store Private CA CRLs** in S3 with appropriate access controls

## 8. Limits

| Resource                                | Limit                            |
| --------------------------------------- | -------------------------------- |
| ACM public certificates per region      | 2,500                            |
| ACM Private CAs per region              | 25                               |
| Domains per certificate                 | 10 (including wildcard)          |
| Imported certificate max size           | 5 KB (PEM)                       |
| Certificate renewal (CloudWatch events) | 5 events (45, 30, 15, 7, 3 days) |

## 9. Exam Tips

1. **ACM public certificates are free** — don't overthink cost for public certs. Only Private CA costs money.

2. **DNS validation** allows automatic renewal. Email validation requires manual action for each renewal.

3. **CloudFront requires ACM in us-east-1** — certificates for CloudFront must be created in the US East (N. Virginia) region.

4. **ACM certificates are regional** — create a certificate in each region where you use it (except CloudFront).

5. **Imported certificates do not auto-renew** — you must re-import before expiry. Monitor expiry events.

6. **ACM Private CA** for internal certificates — issue private certs for internal ALBs, mTLS, IoT, code signing.

7. **Private CA pricing**: ~$400/month per CA + per-certificate fees.

8. **Supported services**: CloudFront, ALB, NLB, API Gateway, Elastic Beanstalk, AppSync, Cognito, Nitro Enclaves.

9. **ACM does not support EC2 instances directly** — ACM certificates can only be used with integrated AWS services.

10. **Certificate validation**: DNS validation (preferred, auto-renew) or Email validation (manual).

11. **ACM + CloudFront**: The certificate must be in us-east-1, and the CloudFront distribution must be associated with it.

12. **ACM + ALB**: Regional certificate, associate with the HTTPS listener.

13. **ACM + NLB**: TLS listeners support ACM certificates (NLB terminates TLS).

14. **Wildcard certificates**: ACM supports `*.example.com` for covering all subdomains.

15. **ACM Private CA can create subordinate CAs** signed by an external root CA — useful for hybrid environments.

16. **Certificate revocation**: Private CA supports CRLs (Certificate Revocation Lists) stored in S3.

17. **ACM Private CA + IAM Roles Anywhere**: Use private certificates to authenticate workloads outside AWS.

18. **CloudWatch events**: Monitor certificate expiry for imported certificates — the events fire at 45, 30, 15, 7, and 3 days.
