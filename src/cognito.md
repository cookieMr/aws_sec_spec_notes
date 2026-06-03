# Amazon Cognito

<figure>
  <img src="./images/Arch_Amazon-Cognito_64@5x.png" alt="Amazon Cognito Icon" width=200>
  <figcaption><center>Amazon Cognito<br><i>Image source: AWS Documentation</i></center></figcaption>
</figure>

**Overview**: Amazon Cognito is AWS's fully managed identity service for web and mobile applications. It handles authentication, authorization, and user management. Cognito is the primary AWS service for adding sign-up, sign-in, and access control to applications without requiring users to have IAM accounts. It is the preferred alternative to calling `AssumeRoleWithWebIdentity` directly.

**Domain weight**: Cognito appears within the IAM domain (~20% of SCS-C03) and across federation, identity, and access control questions throughout the exam.

## 1. Two Core Components

Cognito has two distinct components that are often used together but serve different purposes. The most commonly tested concept on the exam is understanding the difference.

| Component     | Purpose                                         | Output                            | Analogy             |
| ------------- | ----------------------------------------------- | --------------------------------- | ------------------- |
| User Pool     | Authentication — verifies **who** the user is   | JWTs (ID, access, refresh tokens) | A user directory    |
| Identity Pool | Authorization — grants **what** the user can do | Temporary AWS credentials (STS)   | A credential broker |

**Key exam point**: If a question mentions sign-up, sign-in, user management, or tokens → **User Pool**. If it mentions granting temporary AWS credentials to access S3, DynamoDB, or other AWS services → **Identity Pool**. Many real-world scenarios use both together.

## 2. Cognito User Pools

### 2.1. What It Is

- A user directory that provides sign-up and sign-in for application users
- Acts as an OIDC identity provider (IdP) from the application's perspective
- Supports up to 40 million users per pool
- Can be used standalone or integrated with external identity providers

### 2.2. Authentication Methods

| Method                     | Description                                                              |
| -------------------------- | ------------------------------------------------------------------------ |
| Username / Password        | Direct sign-in with Cognito-managed credentials                          |
| Social IdP                 | Google, Facebook, Amazon, Apple — federated sign-in                      |
| SAML 2.0 IdP               | Enterprise identity providers (AD FS, Okta, Ping)                        |
| OIDC IdP                   | Any OIDC-compliant identity provider                                     |
| Custom Authentication Flow | Challenge-based auth using Lambda triggers (biometrics, hardware tokens) |

### 2.3. Token Management

Upon successful authentication, Cognito issues three JSON Web Tokens (JWTs):

| Token             | Purpose                                             | Expiration                     |
| ----------------- | --------------------------------------------------- | ------------------------------ |
| **ID token**      | Contains user identity claims (name, email, phone)  | Configurable (default 1 hour)  |
| **Access token**  | Contains authorization scopes for API access        | Configurable (default 1 hour)  |
| **Refresh token** | Used to obtain new ID/access tokens without re-auth | Longer-lived (default 30 days) |

**Exam points**:

- Tokens are signed (RS256 by default) but **not encrypted** — the payload is base64-encoded JSON that anyone can decode. Encryption of token contents is not a security feature.
- The access token is used with the Cognito Authorizer for API Gateway
- The refresh token is the most sensitive — keep it secure on the client
- Token expiration can be configured per user pool

### 2.4. Multi-Factor Authentication (MFA)

| MFA Type | Description                                      | Security Level |
| -------- | ------------------------------------------------ | -------------- |
| SMS MFA  | One-time code sent via SMS                       | Moderate       |
| TOTP MFA | Time-based one-time password (authenticator app) | High           |

- MFA can be **required** (all users), **optional** (user opt-in), or **adaptive** (enforced based on risk)
- **TOTP is preferred over SMS** due to SMS interception risks (SS7 attacks, SIM swapping)
- Enforced at the User Pool level

### 2.5. Lambda Triggers

Cognito invokes Lambda functions at specific points in the authentication flow. These are commonly tested on the exam.

| Trigger                                 | Timing                       | Common Use Case                              |
| --------------------------------------- | ---------------------------- | -------------------------------------------- |
| **Pre Sign-up**                         | Before user registration     | Validate email domain, auto-confirm users    |
| **Post Confirmation**                   | After user is confirmed      | Onboarding, analytics, welcome email         |
| **Pre Authentication**                  | Before user signs in         | Custom validation, deny list check           |
| **Post Authentication**                 | After user signs in          | Audit logging, analytics                     |
| **Pre Token Generation**                | Before tokens are issued     | Add custom claims to the ID token            |
| **Custom Message**                      | When Cognito sends email/SMS | Customize email/SMS content                  |
| **User Migration**                      | First sign-in attempt        | Migrate users from legacy system on-the-fly  |
| **Define/Create/Verify Auth Challenge** | Custom auth flow             | Implement biometrics, CAPTCHA, hardware keys |

**Exam critical triggers**:

- **Pre Token Generation** — modifying token claims (e.g., adding a `department` claim for ABAC)
- **User Migration** — seamless migration from legacy identity systems; authenticates against the old system on first login
- **Custom Authentication** — non-standard authentication methods (three challenge triggers work together)

### 2.6. Hosted UI

- Cognito provides a pre-built, customizable sign-in/sign-up web page
- Supports OAuth 2.0 grants: authorization code grant, implicit grant, client credentials
- **Best practice for mobile/SPA**: Authorization code grant with PKCE (Proof Key for Code Exchange) — this is more secure than the implicit grant because the authorization code is exchanged for tokens server-side
- Can use a custom domain (e.g., `auth.example.com`) instead of the default domain
- Can be protected with AWS WAF

### 2.7. Groups

- Users can be organized into groups within a User Pool
- Groups can be mapped to IAM roles in Identity Pools for RBAC
- Groups can have precedence values (lower number = higher priority)
- Exam tip: Groups enable role-based access control within Cognito

### 2.8. Resource Servers and Custom Scopes

- Define custom scopes for fine-grained API authorization
- Example: `https://api.example.com/photos.read`, `https://api.example.com/photos.write`
- Scopes are included in the access token
- API Gateway validates scopes via the Cognito Authorizer

## 3. Cognito Identity Pools (Federated Identities)

### 3.1. What It Is

- Grants temporary AWS credentials to users so they can access AWS services directly (S3, DynamoDB, etc.)
- Uses AWS STS internally (`AssumeRoleWithWebIdentity`) to exchange tokens for IAM credentials
- Can work with or without a User Pool

### 3.2. Identity Providers Supported

| Provider Type                      | Description                                     |
| ---------------------------------- | ----------------------------------------------- |
| Cognito User Pool                  | Authenticate via User Pool, get AWS credentials |
| Social IdPs                        | Google, Facebook, Amazon, Apple                 |
| SAML 2.0 IdPs                      | Enterprise federation                           |
| OIDC IdPs                          | Any OIDC-compliant provider                     |
| Developer Authenticated Identities | Custom backend that manages user authentication |

### 3.3. Authenticated vs Unauthenticated (Guest) Access

| Access Type     | Description                                  | IAM Role Scope             |
| --------------- | -------------------------------------------- | -------------------------- |
| Authenticated   | User has proven their identity via an IdP    | Full (but least-privilege) |
| Unauthenticated | Guest/anonymous users with no authentication | Restricted                 |

- Each Identity Pool defines two IAM roles: one for authenticated users, one for unauthenticated (guest) users
- **Exam scenario**: A mobile app needs to let anonymous users upload profile pictures to S3 before signing up → use Identity Pools with an unauthenticated role that has limited S3 put permissions
- Unauthenticated access can be disabled entirely if not needed

### 3.4. IAM Role Mapping

- You can define rules that map users to different IAM roles based on claims in their tokens
- Example: Users from the `admin` Cognito group get an admin IAM role; `viewer` group gets a read-only role
- Role mapping rules are evaluated in order; the first match wins
- Can also use token claims directly in IAM policy variables (`${cognito-identity.amazonaws.com:claims}`)

### 3.5. How Identity Pools Work End-to-End

1. User authenticates with a User Pool (or external IdP) and receives tokens
2. Application passes the token to the Identity Pool
3. Identity Pool validates the token with the IdP
4. Identity Pool calls `AssumeRoleWithWebIdentity` via STS
5. STS returns temporary AWS credentials (AccessKeyId, SecretAccessKey, SessionToken)
6. Application uses these credentials to call AWS services directly
7. Credentials expire automatically (max 1 hour for web identity federation)
8. The SDK automatically refreshes credentials before expiry

## 4. Integration Patterns (Heavily Tested)

### 4.1. API Gateway + Cognito Authorizer (User Pool)

- API Gateway validates the JWT access token before the request reaches the backend
- No custom Lambda authorizer needed — Cognito validates the token natively
- Scopes can be enforced at the API method level
- The backend (Lambda, HTTP, etc.) receives the verified claims in the request context

**Exam scenario**: You have a REST API behind API Gateway. Users authenticate via Cognito. You want to secure the API and pass user identity to the backend → use the **Cognito User Pool Authorizer** on API Gateway.

### 4.2. ALB + Cognito Authentication

- Application Load Balancer can authenticate users via Cognito User Pools before routing traffic
- ALB acts as an OIDC client — it redirects unauthenticated users to the Cognito Hosted UI
- After successful authentication, ALB forwards the user's claims to backend targets via HTTP headers
- No application-level auth code needed — the load balancer handles it

**Exam scenario**: You have a web application behind an ALB. You want to authenticate users at the load balancer level before they reach the application → use **ALB with Cognito User Pool authentication**.

### 4.3. User Pool + Identity Pool (Full Stack)

- User Pool handles sign-up/sign-in and issues tokens
- Identity Pool exchanges tokens for AWS credentials
- Application uses credentials to access S3, DynamoDB, etc.
- This is the most common production architecture

## 5. Advanced Security Features

| Feature                       | Description                                                         |
| ----------------------------- | ------------------------------------------------------------------- |
| Adaptive Authentication       | Evaluates risk (device, location, IP) and adjusts auth requirements |
| Compromised Credentials Check | Checks credentials against known breach databases                   |
| Account Takeover Protection   | Blocks or requires additional verification for suspicious sign-ins  |
| CloudWatch Metrics            | Publishes advanced security metrics for monitoring                  |

- These features must be explicitly enabled and **incur additional cost**
- Available only with the Cognito **Plus** feature plan
- **Exam scenario**: A question mentions detecting risky sign-ins or compromised credentials → answer is **Cognito Advanced Security Features**

## 6. Encryption and Compliance

| Aspect                | Details                                       |
| --------------------- | --------------------------------------------- |
| At-rest encryption    | AES-256 encryption for data stored in Cognito |
| In-transit encryption | TLS for all API communications                |
| Token signing         | RS256 (default) or custom signing keys        |
| Compliance            | HIPAA, SOC, PCI DSS, FedRAMP eligible         |

## 7. Security Best Practices

### 7.1. User Pool Best Practices

- **Enable MFA** — prefer TOTP over SMS
- **Strong password policy** — minimum length, complexity, reuse prevention
- **Use authorization code grant with PKCE** — do not use implicit grant for SPAs and mobile apps
- **Set appropriate token expiration** — shorter for ID/access tokens (15-60 min), longer for refresh tokens (but not excessive)
- **Enable advanced security features** — for risk-based adaptive authentication
- **Use Lambda triggers for custom security logic** — pre sign-up validation, custom message content
- **Protect the Hosted UI with AWS WAF** — rate limiting, IP filtering, bot control
- **Disable unauthenticated access** if guest access is not needed

### 7.2. Identity Pool Best Practices

- **Least privilege IAM roles** — restrict authenticated and unauthenticated roles to minimum required actions
- **Use role mapping rules** — different roles for different user groups
- **Validate tokens server-side** — don't trust client-side claims alone for authorization decisions
- **Enable CloudTrail logging** — audit all `GetId` and `GetCredentialsForIdentity` calls
- **Restrict Identity Pool access** with resource policies where possible

## 8. Limits and Quotas

| Resource                              | Limit              |
| ------------------------------------- | ------------------ |
| Users per User Pool                   | 40 million         |
| User Pools per account                | 1,000 (soft limit) |
| Identity Pools per account            | 1,000 (soft limit) |
| Max ID/access token expiry            | 24 hours           |
| Max refresh token expiry              | 10 years           |
| Max federation metadata document size | 1 MB               |
| Groups per User Pool                  | 10,000             |
| Lambda trigger duration limit         | 5 seconds          |

## 9. Cognito Sync (Legacy)

- Cognito Sync was used to sync user profile data across devices
- **Deprecated in favor of AWS AppSync**
- AppSync with Cognito authorization is the modern replacement
- If the exam mentions cross-device data sync, think **AppSync**, not Cognito Sync

## 10. Exam Tips

1. **User Pool vs Identity Pool** is the #1 Cognito question type on the exam:
   - Sign-up, sign-in, tokens → User Pool
   - Temporary AWS credentials for S3/DynamoDB access → Identity Pool
   - Need both? The tokens from User Pool are fed into Identity Pool for AWS credentials

2. **IAM vs Cognito**: For end-user authentication in web/mobile apps (especially millions of users), the answer is almost always Cognito, not IAM. IAM is for workforce/administrative access.

3. **API Gateway integration**: If securing API Gateway with Cognito → **Cognito User Pool Authorizer** (validates JWTs natively, no Lambda authorizer needed).

4. **ALB integration**: If authenticating users at the load balancer level → **ALB with Cognito as the OIDC IdP**.

5. **Guest access**: If the question mentions anonymous or unauthenticated users accessing AWS resources → **Identity Pool with unauthenticated (guest) role enabled**.

6. **Advanced Security**: If the question mentions compromised credentials, adaptive authentication, risk-based MFA, or account takeover → **Cognito Advanced Security Features** (must be explicitly enabled, extra cost).

7. **Custom auth flow**: If the question describes non-standard authentication (biometrics, CAPTCHA, challenge-response) → **Custom Authentication Flow** using Define/Create/Verify Auth Challenge Lambda triggers.

8. **User migration**: If migrating users from a legacy system → **User Migration Lambda trigger** (authenticates against old system on first login, seamlessly creates Cognito user).

9. **Token customization**: If modifying claims in the JWT → **Pre Token Generation Lambda trigger**.

10. **MFA**: Cognito supports SMS and TOTP. TOTP is preferred over SMS for security. MFA can be required, optional, or adaptive.

11. **Hosted UI security**: Use **authorization code grant with PKCE** (not implicit grant) for mobile and SPA applications.

12. **JWT properties**: Tokens are signed (RS256) but NOT encrypted — anyone can decode the base64 payload. Do not put secrets in tokens.

13. **Scope-based authorization**: Use **Resource Servers** in Cognito to define custom scopes for fine-grained API authorization, enforced by API Gateway.

14. **Groups for RBAC**: Cognito Groups enable role-based access control. Groups can be mapped to IAM roles in Identity Pools.

15. **Cognito does not replace IAM**: Cognito is for application user identities. IAM is for AWS principals (users, roles, services). They serve different purposes and are often used together.

16. **`AssumeRoleWithWebIdentity`**: Identity Pools use this STS API internally to exchange IdP tokens for AWS credentials. The maximum session duration is 1 hour.

17. **CloudTrail**: All Cognito API calls are logged in CloudTrail. Audit `GetId`, `GetCredentialsForIdentity`, `SignUp`, `InitiateAuth`, `AdminInitiateAuth` for security monitoring.

18. **WAF**: The Cognito Hosted UI can be protected by AWS WAF for rate limiting, IP reputation, and bot control.
