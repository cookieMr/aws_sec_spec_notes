# Amazon Detective

<figure>
  <img src="./images/Arch_Amazon-Detective_64@5x.png" alt="Amazon Detective Icon" width=200>
  <figcaption><center>Amazon Detective<br><i>Image source: AWS Documentation</i></center></figcaption>
</figure>

**Overview**: Amazon Detective is an investigation and root cause analysis service that makes it easier to analyze, investigate, and identify the root cause of security findings. Detective automatically collects log data from AWS services and uses machine learning, statistical analysis, and graph theory to build a **behavior graph** that enables security teams to visually navigate and analyze security events.

**Domain weight**: Detective appears in the Threat Detection and Incident Response domain (~14% of SCS-C03). It is the companion to GuardDuty — GuardDuty alerts, Detective investigates.

## 1. What Detective Does

| Capability                  | Description                                                                                 |
| --------------------------- | ------------------------------------------------------------------------------------------- |
| **Pre-computed data model** | Continuously ingests and pre-processes log data into a graph model — no waiting for queries |
| **Behavior graph**          | Visual representation of resource interactions, user activity, and network traffic          |
| **Finding group analysis**  | Correlates related findings into groups for higher-level investigation                      |
| **Root cause analysis**     | Identifies the underlying cause of security findings                                        |
| **Timeline view**           | Presents activity in chronological order for incident reconstruction                        |
| **Visual investigation**    | Interactive graph-based exploration of data                                                 |

## 2. Data Sources

Detective ingests data from multiple AWS services:

| Data Source            | What It Provides                                                    |
| ---------------------- | ------------------------------------------------------------------- |
| **CloudTrail**         | API call history, user identity, source IPs, request details        |
| **VPC Flow Logs**      | Network traffic metadata (IPs, ports, protocols, bytes transferred) |
| **GuardDuty findings** | Detected threats, finding types, severity, affected resources       |
| **EKS audit logs**     | Kubernetes API activity within EKS clusters                         |

- Detective does **not** ingest data from on-premises sources or third-party tools
- Detective processes this data continuously and builds the behavior graph in advance
- **Exam scenario**: A security analyst needs to investigate a GuardDuty finding but does NOT need to write SQL queries — Detective provides a **visual investigation interface** with pre-computed data.

## 3. Behavior Graph

### 3.1. What It Is

- A graph data model that represents **entities** (resources, users, IPs) and **interactions** (API calls, network connections) between them
- Built continuously from ingested data sources
- Enables visual exploration of relationships between entities
- Stored and managed by Detective — no manual construction needed

### 3.2. How It Helps Investigation

Instead of running multiple queries across CloudTrail, VPC Flow Logs, and GuardDuty, an analyst can:

1. Start from a GuardDuty finding
2. Click on the affected EC2 instance
3. See all API calls made by the instance
4. See all network connections to/from the instance
5. See which IAM role the instance was using
6. See what actions that role performed

All of this is pre-computed and available instantly through the visual interface.

## 4. Finding Groups

- Detective automatically groups related findings into **finding groups**
- A finding group represents a security event that may involve multiple related findings
- Example: A compromised EC2 instance might generate findings from GuardDuty (malicious IP communication), VPC Flow Logs (data exfiltration), and CloudTrail (unusual IAM role usage) — Detective groups these together
- Finding groups include a **severity score**, a **scope** (affected resources), and a **timeframe**

**Exam scenario**: Multiple GuardDuty findings appear related to the same incident, but the analyst needs to see them as a single investigation → use **Detective finding groups** which automatically correlate related findings.

## 5. Integration with Other Services

| Service               | Integration                                                                               |
| --------------------- | ----------------------------------------------------------------------------------------- |
| **GuardDuty**         | Findings are automatically ingested — one-click investigation from GuardDuty to Detective |
| **Security Hub**      | Findings from Security Hub can be investigated in Detective                               |
| **CloudTrail**        | API activity data is ingested for behavioral analysis                                     |
| **VPC Flow Logs**     | Network traffic data is ingested for analysis                                             |
| **EKS**               | Kubernetes audit logs are ingested for analysis                                           |
| **AWS Organizations** | Delegated administrator for multi-account investigation                                   |

### 5.1. GuardDuty + Detective

This is the most important integration on the exam:

- GuardDuty **detects** threats and generates findings
- Detective **investigates** those findings and provides root cause analysis
- From GuardDuty, you can click "Investigate" to open Detective with the finding context pre-loaded

**Exam scenario**: An analyst receives a GuardDuty alert about a potentially compromised EC2 instance. They need to understand what happened before and after the alert → use **Detective** to investigate the finding's full context (related API calls, network traffic, user activity).

### 5.2. Detective + Security Hub

- Security Hub aggregates findings from GuardDuty, Inspector, Macie, etc.
- From Security Hub, analysts can pivot into Detective for investigation
- Detective provides the detailed behavioral context missing from Security Hub's summary view

## 6. Multi-Account Management

- Designate a **delegated administrator** from the management account of AWS Organizations
- The delegated admin can enable Detective for member accounts
- The behavior graph spans all enabled accounts — no need to switch between accounts for investigation
- Member accounts can manage their own Detective data if not restricted

**Exam scenario**: A security team investigates a cross-account attack — the behavior graph in the delegated admin account shows activity across all member accounts in a single view.

## 7. Cost

| Cost Driver               | Details                                                              |
| ------------------------- | -------------------------------------------------------------------- |
| **Data ingestion**        | Charged per GB of data ingested (CloudTrail, VPC Flow Logs, etc.)    |
| **Data retention**        | Included in ingestion cost (data is retained for the behavior graph) |
| **Administrator account** | Pays for the organization's Detective costs                          |

- Detective retains data for the lifetime of the behavior graph
- Deleting the behavior graph removes all data
- Detective is typically used on-demand during investigations rather than kept running constantly (though it needs to run to build the graph)

## 8. Security Best Practices

- **Enable Detective in the security account** for centralized investigation
- **Integrate with GuardDuty** — Detective is most valuable when GuardDuty is producing findings to investigate
- **Retain Detective data** for the incident response window required by compliance
- **Use organization-level delegation** for multi-account visibility
- **Ensure CloudTrail and VPC Flow Logs** are enabled — Detective needs these data sources

## 9. Limits and Quotas

| Resource                           | Limit                                                   |
| ---------------------------------- | ------------------------------------------------------- |
| Behavior graphs per account        | 1 per region                                            |
| Data retention                     | Retention of behavior graph (deletion removes all data) |
| Member accounts per behavior graph | 1,200 per region                                        |
| Finding groups                     | Unlimited                                               |

## 10. Detective vs Other Services

| Service           | When to Use                                |
| ----------------- | ------------------------------------------ |
| **GuardDuty**     | Alert me when something suspicious happens |
| **Detective**     | Investigate what happened and why          |
| **CloudTrail**    | Find specific API calls by a specific user |
| **VPC Flow Logs** | Analyze network traffic patterns           |
| **Security Hub**  | View all findings in one place             |
| **Athena**        | Run custom SQL queries on S3 logs          |

**Exam tip**: Detective is the **investigation** step in the incident response workflow. GuardDuty detects, Security Hub aggregates, Detective investigates.

## 11. Exam Tips

1. **Detective vs GuardDuty**: GuardDuty **detects** threats (generates findings). Detective **investigates** findings (provides root cause analysis). They are often used together.

2. **Detective provides visual investigation** — it is NOT a query service. You navigate the behavior graph visually rather than writing SQL queries.

3. **Data sources**: CloudTrail, VPC Flow Logs, GuardDuty findings, EKS audit logs. Detective cannot ingest third-party data or on-premises logs.

4. **Behavior graph** is pre-computed — data is processed and ready for analysis immediately. There is no waiting for queries to run.

5. **Finding groups** automatically correlate related findings into a single incident — reduces noise and provides context.

6. **Multi-account**: Use a delegated administrator for cross-account investigation without switching consoles.

7. **GuardDuty → Detective**: The one-click "Investigate" button in GuardDuty opens Detective with full finding context — this is the primary workflow.

8. **Detective does not prevent or stop attacks** — it helps you understand what happened after an attack is detected.

9. **Detective is not a SIEM** — it is a purpose-built investigation tool for AWS security findings. Do not confuse it with third-party SIEM solutions.

10. **Cost**: Pay per GB of ingested data. Typical usage is to have Detective running continuously or enable it on-demand during active investigations.

11. **Data retention**: As long as the behavior graph exists, data is retained. Deleting the graph removes all data.

12. **One behavior graph per region** — data is regional but can span multiple accounts within that region via delegated administration.

13. **Complementary to Athena**: Athena is better for ad-hoc SQL queries on raw logs. Detective is better for visual exploration of entity relationships.

14. **Security Hub integration**: You can pivot from Security Hub findings directly into Detective for investigation.

15. **Ideal for root cause analysis**: When a question asks "how to determine the root cause of a security finding" or "how to investigate what led to a compromise", the answer is Detective.
