# Amazon VPC

<figure>
  <img src="./images/amazon-vpc.png" alt="Amazon VPC Icon" width=200>
  <figcaption><center>Amazon VPC<br><i>Image source: <a href="https://vecta.io/symbols/9/aws-compute/19/amazon-vpc">Vecta.io</a></i></center></figcaption>
</figure>

**Overview**: Amazon Virtual Private Cloud (VPC) is the networking layer for AWS resources. VPC provides isolation, connectivity, and traffic control. For the Security Specialty exam, understanding VPC security mechanisms — security groups, NACLs, VPC endpoints, Flow Logs, and network segmentation — is essential. VPC is the foundation of infrastructure security in AWS.

**Domain weight**: VPC is central to the Infrastructure Security domain (~20% of SCS-C03), the single largest domain. VPC security appears in questions about network isolation, traffic filtering, private connectivity, and logging.

## 1. VPC Core Components

| Component            | Purpose                                                              |
| -------------------- | -------------------------------------------------------------------- |
| **VPC**              | Isolated virtual network within a region                             |
| **Subnet**           | A range of IP addresses within a VPC, tied to one AZ                 |
| **Route table**      | Controls where network traffic is directed                           |
| **Internet Gateway** | Enables internet access for public subnets                           |
| **NAT Gateway**      | Enables outbound internet from private subnets                       |
| **Security Group**   | Instance-level stateful firewall (allow rules only)                  |
| **Network ACL**      | Subnet-level stateless firewall (allow/deny rules)                   |
| **VPC Endpoint**     | Private connectivity to AWS services without internet                |
| **Network Firewall** | Managed stateful firewall for VPC-to-VPC and VPC-to-internet traffic |

## 2. Security Groups

### 2.1. Key Characteristics

- **Stateful**: Return traffic is automatically allowed regardless of outbound rules
- **Allow rules only**: Cannot explicitly deny traffic (implicit deny for everything else)
- **Per-instance**: Applied at the ENI (Elastic Network Interface) level
- **Evaluate all rules**: All rules in a security group are evaluated together
- **Default**: VPC comes with a default security group that allows all inbound traffic from within the group

### 2.2. Security Group vs NACL

| Feature                 | Security Group       | Network ACL                   |
| ----------------------- | -------------------- | ----------------------------- |
| **Scope**               | Instance-level (ENI) | Subnet-level                  |
| **State**               | Stateful             | Stateless                     |
| **Rules**               | Allow only           | Allow and Deny                |
| **Evaluation order**    | All rules evaluated  | Numbered order (lowest first) |
| **Supports deny rules** | No                   | Yes                           |
| **Applies to**          | Specific resources   | Entire subnet                 |

### 2.3. Common Exam Scenarios

- **EC2 can't connect to the internet**: Security group outbound rules blocking traffic (or no IGW/NAT in route table)
- **EC2 can't be reached from the internet**: Security group inbound rules not allowing the traffic, or instance is in a private subnet
- **Two EC2 instances can't communicate**: Check both security groups — each must allow traffic from the other
- **Security group as a source**: You can reference another security group as the CIDR source, enabling dynamic allow-rules between instances in different groups

## 3. Network ACLs (NACLs)

### 3.1. Key Characteristics

- **Stateless**: Return traffic must be explicitly allowed by a separate outbound rule
- **Numbered rules**: Evaluated from lowest number to highest; first match wins
- **Supports allow and deny**: Can explicitly deny specific traffic
- **Subnet-level**: Applies to all resources in the subnet
- **Default NACL**: Allows all inbound and outbound traffic

### 3.2. Stateless Behavior (Exam Critical)

Because NACLs are stateless, you must configure both inbound and outbound rules for traffic to flow:

| Direction    | Source    | Destination                           | Rule                 |
| ------------ | --------- | ------------------------------------- | -------------------- |
| **Inbound**  | Client IP | EC2 (ephemeral port range 1024-65535) | Allow HTTP/HTTPS     |
| **Outbound** | EC2       | Client IP (ephemeral port)            | Allow return traffic |

**Exam scenario**: Web servers in a public subnet receive traffic from the internet, but responses are blocked → the **outbound NACL rule** is missing to allow the return traffic (NACLs are stateless). Security groups would automatically allow this response traffic (stateful).

### 3.3. NACL Ephemeral Ports

- Clients use random ports (1024-65535) for return traffic
- NACL outbound rules must allow traffic to these ephemeral ports
- Common rule: Allow outbound traffic to 1024-65535 for return traffic

## 4. VPC Endpoints

### 4.1. Gateway Endpoints vs Interface Endpoints

| Feature                | Gateway Endpoint        | Interface Endpoint           |
| ---------------------- | ----------------------- | ---------------------------- |
| **Services**           | S3, DynamoDB            | 70+ AWS services             |
| **Technology**         | Gateway in route table  | ENI (powered by PrivateLink) |
| **Cost**               | Free                    | Per hour + per GB processed  |
| **Access**             | Via route table entries | Via DNS or security group    |
| **On-premises access** | No                      | Yes (via VPN/Direct Connect) |
| **Cross-region**       | No                      | No                           |

### 4.2. Gateway Endpoint (S3, DynamoDB)

- Adds a prefix list in the route table (no IGW or NAT needed)
- **Free** — no additional charge
- Access is controlled via **endpoint policies** (resource-based policies)
- Traffic stays within AWS network — does not go through the internet
- Only works from within the same region

**Exam scenario**: EC2 instances in a private subnet need to access S3 without going through a NAT Gateway → create a **VPC Gateway Endpoint** for S3 and add a route to the prefix list.

### 4.3. Interface Endpoint (PrivateLink)

- Creates an Elastic Network Interface (ENI) in the subnet with a private IP
- Access is via DNS or security group
- Supports most AWS services (SNS, SQS, KMS, CloudWatch, Lambda, etc.)
- Used for **PrivateLink** — connecting services across VPCs and accounts
- **Exam scenario**: A Lambda function in a VPC needs to send messages to SQS without internet access → create a **VPC Interface Endpoint** for SQS (PrivateLink).

### 4.4. Endpoint Policies

- JSON policies that control which actions principals can perform through the endpoint
- Example: Restrict the S3 Gateway Endpoint to allow only `GetObject` from a specific bucket
- Separate from IAM policies — both must allow the action

```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-secure-bucket/*"
    }
  ]
}
```

## 5. VPC Flow Logs

### 5.1. Purpose

- Captures metadata about network traffic in the VPC
- Logs source IP, destination IP, ports, protocol, action (ACCEPT/REJECT), packet count, byte count
- Can be published to **CloudWatch Logs** or **S3**

### 5.2. Levels

| Level      | Captures Traffic From        |
| ---------- | ---------------------------- |
| **VPC**    | All ENIs in the VPC          |
| **Subnet** | All ENIs in the subnet       |
| **ENI**    | A specific network interface |

### 5.3. Fields

Key fields in a VPC Flow Log record:

| Field        | Description                          |
| ------------ | ------------------------------------ |
| `srcaddr`    | Source IP address                    |
| `dstaddr`    | Destination IP address               |
| `srcport`    | Source port                          |
| `dstport`    | Destination port                     |
| `protocol`   | IANA protocol number (6=TCP, 17=UDP) |
| `action`     | ACCEPT or REJECT                     |
| `log-status` | OK, NODATA, SKIPDATA                 |

### 5.4. Security Use Cases

- Detect **rejected connections** (potential port scanning)
- Identify **unusual traffic patterns** (data exfiltration)
- Detect **traffic to known-bad IPs** (with Athena or CloudWatch metric filters)
- Troubleshoot connectivity issues (is traffic being accepted or rejected? By which rule?)

**Exam scenario**: An administrator needs to determine why an EC2 instance cannot receive traffic on port 443 → check **VPC Flow Logs** to see whether the traffic is being ACCEPTED or REJECTED. If REJECTED, check security groups and NACLs.

### 5.5. VPC Flow Logs vs CloudTrail

| Service       | Logs                                   |
| ------------- | -------------------------------------- |
| CloudTrail    | API calls (who, what, when)            |
| VPC Flow Logs | Network traffic (IPs, ports, protocol) |

## 6. PrivateLink

### 6.1. Purpose

- Expose a service (running behind a NLB) privately across VPCs and accounts
- The consumer creates an **Interface Endpoint** (ENI) in their VPC
- The provider publishes the service via a **NLB (Network Load Balancer)** and a **VPC Endpoint Service**
- Traffic stays within AWS network — no internet, no VPC peering, no VPN

**Exam scenario**: A security team in Account A has a security monitoring tool. Account B wants to send logs through it privately → Account A creates a **NLB + VPC Endpoint Service**. Account B creates an **Interface Endpoint** to connect privately.

### 6.2. Key Points

- Usable across accounts, VPCs, and regions
- The consumer controls security groups on their ENI
- The provider controls the NLB's security
- No VPC peering or Transit Gateway required
- Supports TCP only (NLB requirement)

### 6.3. PrivateLink vs VPC Peering

| Feature                | PrivateLink                 | VPC Peering                 |
| ---------------------- | --------------------------- | --------------------------- |
| **Traffic scope**      | Single service only         | Full VPC connectivity       |
| **Network complexity** | No overlapping CIDRs issues | No overlapping CIDRs issues |
| **Transitive**         | No                          | No (star topology required) |
| **Cross-account**      | Yes                         | Yes                         |
| **Cross-region**       | Yes                         | Yes                         |
| **Bandwidth**          | Per-connection (NLB limits) | Full bandwidth              |

## 7. VPC Peering

- Connects two VPCs (same or different accounts/regions) using private IP addresses
- **Not transitive** — if VPC A is peered with VPC B and VPC B is peered with VPC C, VPC A cannot reach VPC C
- Route tables must be updated manually in each VPC
- No overlapping CIDR ranges allowed
- Cross-region peering is available

**Exam scenario**: A company has VPCs in two accounts that need full private IP connectivity → use **VPC Peering** (if only a few VPCs). For many VPCs, use **Transit Gateway**.

## 8. Transit Gateway

- A hub-and-spoke model for connecting many VPCs and on-premises networks
- Supports transitive routing — connected VPCs can reach each other through the Transit Gateway
- Supports **route tables** for segmentation (isolate environments)
- Supports **multicast** (Transit Gateway Multicast)
- Supports cross-account attachments

**Exam scenario**: A company has 50 VPCs across multiple accounts and regions that all need to communicate → use **Transit Gateway** for a hub-and-spoke topology (VPC Peering is not scalable for 50 VPCs).

## 9. AWS Network Firewall

### 9.1. Purpose

- A managed stateful firewall service for VPCs
- Inspects traffic: VPC-to-VPC, VPC-to-internet, internet-to-VPC, Direct Connect, VPN
- Supports: stateful inspection, domain filtering, intrusion prevention (Suricata-based), threat intelligence

### 9.2. Key Features

- **Stateful rules** (unlike NACLs which are stateless)
- **Domain filtering**: Allow/block traffic to specific domains (FQBNs)
- **IP reputation**: Block traffic from known-bad IPs
- **Intrusion prevention**: Suricata-compatible rules for threat detection
- **TLS inspection**: Decrypt and inspect TLS traffic (with appropriate certificates)

### 9.3. Network Firewall vs Security Groups vs NACLs

| Layer                | Scope    | State     | Complexity |
| -------------------- | -------- | --------- | ---------- |
| **Security Groups**  | Instance | Stateful  | Simple     |
| **NACLs**            | Subnet   | Stateless | Medium     |
| **Network Firewall** | VPC      | Stateful  | Advanced   |

### 9.4. Exam Scenarios

- **Domain filtering**: Block outbound traffic to known-bad domains or allow only approved domains
- **Threat prevention**: Suricata-based IPS rules to detect and block C2 traffic
- **Inbound filtering**: Advanced traffic inspection beyond security groups and NACLs

## 10. Subnet Types and Traffic Flow

### 10.1. Public Subnet

- Route table has a route to an Internet Gateway (IGW)
- Instances can have public IPs (direct internet access)
- Resources need security group rules to allow inbound internet traffic

### 10.2. Private Subnet

- No direct route to an IGW
- Outbound internet via **NAT Gateway** (in a public subnet)
- Inbound from internet only via **ALB/NLB in a public subnet** or **VPN/Direct Connect**

### 10.3. VPN-Only Subnet

- Route table routes traffic through a **Virtual Private Gateway** (VPG)
- Used for hybrid cloud extensions of on-premises networks

## 11. Encryption in Transit

| Scenario                 | Solution                                                                |
| ------------------------ | ----------------------------------------------------------------------- |
| VPC to VPC               | VPC Peering or Transit Gateway (AWS backbone — automatically encrypted) |
| EC2 to EC2 (same VPC)    | No encryption by default (use application-level TLS)                    |
| EC2 to S3 (via endpoint) | AWS backbone — no internet. Use HTTPS for application-level encryption  |
| VPN to on-premises       | IPsec tunnels                                                           |
| Direct Connect           | No encryption by default (optionally add VPN over DX for encryption)    |

## 12. Security Best Practices

- **Use security groups as the primary firewall** — they are simpler and more flexible than NACLs
- **Use NACLs as a secondary layer** — add explicit deny for known-bad IPs or broad subnet-level rules
- **Enable VPC Flow Logs** in all VPCs (VPC, subnet, or ENI level)
- **Use VPC endpoints** instead of NAT Gateway + internet for AWS service access
- **Deploy resources in private subnets** — only load balancers and bastions in public subnets
- **Use separate VPCs for different environments** (prod, dev, test) with separate security controls
- **Use Network Firewall** for advanced inspection (domain filtering, IPS, TLS inspection)
- **Use PrivateLink** for cross-account service access instead of exposing services to the internet
- **Do not use 0.0.0.0/0 in security groups** unless absolutely necessary (prefer specific CIDRs)
- **Monitor security group and NACL changes** via CloudTrail
- **Limit SSH/RDP access** to a bastion host or VPN — never expose directly to the internet

## 13. Limits and Quotas

| Resource                        | Limit                            |
| ------------------------------- | -------------------------------- |
| VPCs per region per account     | 5 (soft limit)                   |
| Subnets per VPC                 | 200                              |
| Security groups per VPC         | 2,500                            |
| Rules per security group        | 60 (inbound + outbound combined) |
| NACL rules per NACL             | 20 (inbound + 20 outbound)       |
| VPC peering connections per VPC | 125                              |
| Gateway endpoints per VPC       | 255                              |
| Interface endpoints per VPC     | 255                              |

## 14. Exam Tips

1. **Security groups are stateful** — return traffic is automatically allowed. **NACLs are stateless** — both inbound AND outbound rules must be explicitly set for traffic to flow.

2. **Security groups are allow-only** — you cannot write a deny rule. Use NACLs for explicit deny (e.g., block traffic from a specific IP).

3. **NACL rule evaluation** — lowest number wins. Place broad allow rules at low numbers and deny rules at higher numbers.

4. **Ephemeral ports**: NACL outbound rules must allow return traffic on ephemeral ports (1024-65535) because NACLs are stateless.

5. **VPC Flow Logs** capture network metadata (ACCEPT/REJECT) — use them to debug connectivity, not CloudTrail.

6. **Gateway Endpoints (S3, DynamoDB)**: Free, accessed via route table, no internet needed. **Interface Endpoints (PrivateLink)**: Paid, accessed via ENI + DNS, supports most AWS services.

7. **VPC Peering is not transitive** — if you need transitive routing, use Transit Gateway.

8. **PrivateLink** connects services across VPCs/accounts without VPC peering. The provider uses NLB + Endpoint Service. The consumer creates an Interface Endpoint.

9. **Network Firewall**: For advanced inspection — domain filtering, Suricata-based IPS, TLS inspection. Goes beyond security groups and NACLs.

10. **NAT Gateway**: Provides outbound internet for private subnets. Placed in a public subnet. Charged by hour and data processed.

11. **Internet Gateway**: Provides both inbound and outbound internet for public subnets. Scales automatically.

12. **DNS resolution**: Enable both DNS resolution and DNS hostnames for the VPC to use private DNS with VPC endpoints.

13. **Endpoint policies** control what actions are allowed through a VPC endpoint — they are resource-based policies evaluated alongside IAM.

14. **Default VPC**: Every new account gets a default VPC in each region with a public subnet, IGW, and default security group. For production, create custom VPCs.

15. **Overlapping CIDRs**: Cannot be used with VPC Peering or VPN connections. Use distinct CIDR ranges for each VPC.
