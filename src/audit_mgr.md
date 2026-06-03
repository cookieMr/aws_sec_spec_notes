# AWS Audit Manager

<figure>
  <img src="./images/Arch_AWS-Audit-Manager_64@5x.png" alt="AWS Audit Manager Icon" width=200>
  <figcaption><center>AWS Audit Manager<br><i>Image source: AWS Documentation</i></center></figcaption>
</figure>

**Overview**: AWS Audit Manager helps continuously audit AWS usage to simplify risk and compliance assessment. It automatically collects evidence from AWS services (Config, CloudTrail, Security Hub, etc.) and maps it to **control frameworks** (SOC 2, PCI DSS, HIPAA, GDPR, ISO 27001, etc.) to reduce the manual effort of audit preparation. It provides a single place to manage assessments, collect evidence, and generate audit-ready reports.

**Domain weight**: Appears in Management & Governance. Audit Manager is a **niche service** — typically 0–1 questions on the exam. Questions are straightforward: "Which service automatically collects audit evidence and maps it to compliance controls?"

---

## 1. Core Concepts

### 1.1. Assessment Frameworks

- **Prebuilt frameworks** for common compliance standards

| Framework                                    | Type     | Description                                                      |
| -------------------------------------------- | -------- | ---------------------------------------------------------------- |
| **SOC 2**                                    | Prebuilt | Trust Service Criteria (security, availability, confidentiality) |
| **PCI DSS v3.2.1**                           | Prebuilt | Payment Card Industry Data Security Standard                     |
| **HIPAA**                                    | Prebuilt | Healthcare privacy and security rules                            |
| **GDPR**                                     | Prebuilt | EU General Data Protection Regulation                            |
| **ISO 27001:2013**                           | Prebuilt | Information security management                                  |
| **AWS Foundational Security Best Practices** | Prebuilt | AWS Well-Architected security pillar                             |
| **Custom**                                   | Custom   | Create your own framework with custom controls                   |

- Each framework has a set of **controls** mapped to specific **evidence sources**
- Controls can be customized: add/remove evidence sources, modify test procedures

### 1.2. Assessments

- An **assessment** is an instance of a framework applied to a specific scope (accounts, regions, resources)
- Assessments contain:
  - **Framework**: The compliance standard being assessed
  - **Scope**: Which AWS accounts and resources are in scope
  - **Controls**: The specific requirements being tested
  - **Evidence**: Automated evidence collected from AWS services + manual evidence uploads
  - **Reports**: Assessment reports for auditors

### 1.3. Evidence

| Evidence Type         | Description                                                                        |
| --------------------- | ---------------------------------------------------------------------------------- |
| **Automated**         | Collected automatically from AWS services (Config, CloudTrail, Security Hub, etc.) |
| **Manual**            | Uploaded by the user (PDFs, screenshots, policy documents)                         |
| **Findings**          | From Security Hub, GuardDuty, Inspector, Macie                                     |
| **Compliance checks** | From AWS Config rules                                                              |

### 1.4. Evidence Sources

| AWS Service      | Evidence Collected                                                       |
| ---------------- | ------------------------------------------------------------------------ |
| **AWS Config**   | Resource configuration history, compliance rules, resource relationships |
| **CloudTrail**   | API activity logs (who did what, when)                                   |
| **Security Hub** | Security findings, compliance standards                                  |
| **GuardDuty**    | Threat detection findings                                                |
| **Inspector**    | Vulnerability assessment findings                                        |
| **IAM**          | Access Key last used, policy summaries                                   |
| **S3**           | Bucket policies, encryption settings, public access settings             |
| **KMS**          | Key policies, rotation status                                            |

## 2. Assessment Workflow

### 2.1. Step-by-Step

1. **Choose a framework**: Select a prebuilt framework (SOC 2, PCI DSS, HIPAA, etc.) or create a custom one
2. **Define the scope**: Select AWS accounts and resources to be assessed
3. **Configure controls**: Customize control mappings and evidence sources
4. **Run the assessment**: Audit Manager automatically collects evidence based on the schedule
5. **Review evidence**: Validate that the collected evidence demonstrates compliance
6. **Generate reports**: Create assessment reports for stakeholders and auditors
7. **Delegate to auditor**: Share the assessment with your auditor for review

### 2.2. Assessment Schedule

- Assessments can run **on-demand** or on a **schedule** (daily, weekly, or monthly)
- Evidence collection is continuous — new evidence is appended between assessment runs
- Schedule aligned with the compliance reporting period (e.g., quarterly SOC review)

## 3. Audit Reports

### 3.1. Report Types

| Report Type           | Contents                                             | Use Case                             |
| --------------------- | ---------------------------------------------------- | ------------------------------------ |
| **Assessment report** | Control status, evidence summary, compliance score   | Internal review of compliance status |
| **Evidence report**   | Detailed evidence for each control (with timestamps) | Sharing with auditors                |
| **Snapshot**          | Point-in-time view of the assessment report          | Auditor review at a specific date    |

### 3.2. Auditor Delegation

- Assessment reports can be **shared with auditors** without giving them AWS account access
- An auditor receives a **shareable link** to view the assessment
- The auditor can view: control status, evidence, compliance scores
- The auditor can download evidence and reports
- No IAM role or AWS account required for the auditor

**Exam scenario**: An external auditor needs to review evidence for a SOC 2 assessment without accessing the AWS account → use **Audit Manager** to generate a report and delegate the assessment to the auditor via a shareable link.

## 4. Custom Frameworks

- Create **custom frameworks** with your own controls and evidence sources
- Define: control name, description, test procedure, evidence mappings
- Import/export frameworks as JSON for sharing across accounts
- Customize prebuilt frameworks by adding or removing controls

**Exam scenario**: An organization has internal compliance requirements that go beyond standard frameworks (SOC 2, HIPAA) → create a **custom framework** in AWS Audit Manager with the additional controls.

## 5. Integration with AWS Services

| Service                | Integration                                                    |
| ---------------------- | -------------------------------------------------------------- |
| **AWS Organizations**  | Centrally manage assessments across all accounts               |
| **AWS Config**         | Primary evidence source for resource configurations            |
| **AWS CloudTrail**     | Evidence source for API activity                               |
| **AWS Security Hub**   | Evidence source for security findings and compliance standards |
| **Amazon GuardDuty**   | Evidence source for threat detection findings                  |
| **Amazon Inspector**   | Evidence source for vulnerability scan findings                |
| **Amazon S3**          | Stores evidence snapshots and reports                          |
| **AWS KMS**            | Encrypts assessment data                                       |
| **Amazon EventBridge** | Notification events for assessment lifecycle                   |
| **AWS CloudFormation** | Deploy Audit Manager resources as infrastructure as code       |

## 6. Access Control

### 6.1. IAM Permissions

| Action                                | Description                            |
| ------------------------------------- | -------------------------------------- |
| `auditmanager:CreateAssessment`       | Create a new assessment                |
| `auditmanager:UpdateAssessment`       | Update assessment settings             |
| `auditmanager:GetAssessment`          | Get assessment details and evidence    |
| `auditmanager:DeleteAssessment`       | Delete an assessment                   |
| `auditmanager:CreateAssessmentReport` | Generate an assessment report          |
| `auditmanager:GetEvidence`            | Retrieve evidence for a control        |
| `auditmanager:GetChangeLogs`          | View change logs for evidence/controls |
| `auditmanager:RegisterAccount`        | Enable Audit Manager for the account   |

### 6.2. Organizations Delegated Admin

- A **delegated administrator** can manage Audit Manager across the entire AWS Organization
- The delegated admin creates assessments that cover all member accounts
- Simplifies multi-account compliance management
- Set via: AWS Organizations → Register delegated administrator for Audit Manager

## 7. Common Exam Scenarios

1. **Audit evidence collection**: A compliance team needs to automatically collect evidence for a SOC 2 audit → use **AWS Audit Manager** with the prebuilt SOC 2 framework.

2. **Auditor access without AWS account**: An external auditor needs to review compliance evidence without signing into the AWS account → generate an **assessment report** and **delegate** it to the auditor.

3. **Multi-account compliance**: An organization with multiple accounts needs a unified view of compliance → use a **delegated administrator** in Audit Manager to manage assessments across all accounts.

4. **Custom compliance framework**: An organization has unique internal compliance controls → create a **custom framework** in Audit Manager.

5. **Continuous compliance monitoring**: A security team wants ongoing visibility into compliance status → set up **scheduled assessments** in Audit Manager with periodic evidence collection.

## 8. Audit Manager vs Other Services

| Feature                 | AWS Audit Manager                        | AWS Config                               | AWS Security Hub                                       |
| ----------------------- | ---------------------------------------- | ---------------------------------------- | ------------------------------------------------------ |
| **Purpose**             | Audit evidence collection & reports      | Resource configuration monitoring        | Security findings aggregation                          |
| **Frameworks**          | SOC 2, PCI DSS, HIPAA, GDPR, ISO, custom | AWS managed Config rules                 | AWS Foundational Security Best Practices, CIS, PCI DSS |
| **Output**              | Audit reports, evidence packages         | Compliance scores, configuration history | Security scores, findings                              |
| **Auditor access**      | ✅ Delegate to external auditors         | ❌ No auditor access                     | ❌ No auditor access                                   |
| **Evidence collection** | ✅ Automatic from multiple sources       | ✅ Resource configuration only           | ✅ Security findings only                              |
| **Best for**            | Full audit lifecycle management          | Resource compliance monitoring           | Security posture management                            |

**Exam scenario**: A compliance team needs to prepare for an external audit with SOC 2 evidence, control mappings, and auditor-ready reports → use **AWS Audit Manager** (not just Config or Security Hub alone).

## 9. Exam Tips

1. **Audit Manager = automate audit evidence**: It collects evidence from Config, CloudTrail, Security Hub, GuardDuty, Inspector — all mapped to compliance controls.

2. **Prebuilt frameworks**: SOC 2, PCI DSS, HIPAA, GDPR, ISO 27001 — the exam may ask which service has these built-in.

3. **Auditor delegation**: Share assessment reports with external auditors via a link — no AWS account needed.

4. **Custom frameworks**: If the prebuilt frameworks do not match the organization's needs, create a custom one.

5. **Delegated administrator**: Manage audits across all accounts in an AWS Organization from a single account.

6. **Evidence sources**: Config (resource configs), CloudTrail (API calls), Security Hub (findings), GuardDuty (threats), Inspector (vulnerabilities).

7. **Not a real-time security tool**: Audit Manager is for compliance evidence — it does not detect threats or enforce policies.

8. **Scheduled assessments**: Evidence collection runs on a schedule (daily, weekly, monthly) — not real-time.

9. **Snapshot reports**: Point-in-time reports for auditor review — capture compliance status at a specific date.

10. **Audit Manager vs Config**: Config tracks individual resource compliance. Audit Manager aggregates evidence across multiple services and maps it to audit controls.
