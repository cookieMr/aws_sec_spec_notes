# Amazon Athena

<figure>
  <img src="./images/Arch_Amazon-Athena_64@5x.png" alt="Amazon Athena Icon" width=200>
  <figcaption><center>Amazon Athena<br><i>Image source: AWS Documentation</i></center></figcaption>
</figure>

**Overview**: Amazon Athena is an interactive query service that lets you analyze data stored in Amazon S3 using standard SQL. It is serverless — no infrastructure to manage, and you pay only for the data scanned by your queries. For the Security Specialty exam, Athena is the primary tool for ad-hoc analysis of security logs stored in S3, particularly CloudTrail logs, VPC Flow Logs, and AWS Config data.

**Domain weight**: Athena appears in the Security Logging and Monitoring domain (~18% of SCS-C03) and the Data Protection domain. It is the standard answer for any scenario that requires querying S3-stored logs with SQL.

## 1. How Athena Works

1. Data is stored in S3 (CloudTrail logs, VPC Flow Logs, etc.)
2. A table schema is defined in the **AWS Glue Data Catalog** (or manually)
3. You write standard SQL queries in the Athena console, CLI, or API
4. Athena reads the data from S3, executes the query, and returns results
5. You pay only for the amount of data scanned

| Component             | Description                                                     |
| --------------------- | --------------------------------------------------------------- |
| **Data source**       | S3 bucket containing the log data                               |
| **Glue Data Catalog** | Stores table definitions (schema, partitions, location)         |
| **Query engine**      | Presto-based distributed SQL engine (serverless)                |
| **Output**            | Query results stored in a separate S3 bucket (specified by you) |

## 2. Security Use Cases

### 2.1. CloudTrail Log Analysis

Athena is commonly used to query CloudTrail logs stored in S3:

```sql
-- Find all ConsoleLogin events in the last 7 days
SELECT
  useridentity.arn,
  useridentity.type,
  eventname,
  sourceipaddress,
  eventtime
FROM
  cloudtrail_logs
WHERE
  eventname = 'ConsoleLogin'
  AND eventtime >= date_add('day', -7, now())
ORDER BY
  eventtime DESC;
```

**Exam scenario**: A security analyst needs to find all `DeleteBucket` API calls across the last 90 days → use **Athena** to query CloudTrail logs stored in S3.

### 2.2. VPC Flow Log Analysis

```sql
-- Find top 10 source IPs by connection count
SELECT
  srcaddr,
  dstaddr,
  dstport,
  COUNT(*) AS connection_count
FROM
  vpc_flow_logs
WHERE
  action = 'REJECT'
GROUP BY
  srcaddr,
  dstaddr,
  dstport
ORDER BY
  connection_count DESC
LIMIT 10;
```

### 2.3. AWS Config Queries

- Query resource compliance history stored by AWS Config in S3
- Example: Find all S3 buckets that were non-compliant with encryption rules

### 2.4. Security Lake Queries

- When Security Lake is used, Athena queries OCSF-normalized Parquet data
- Tables are automatically populated in the Glue Data Catalog by Security Lake
- Example: Query CloudTrail and VPC Flow Logs together using the normalized OCSF schema

## 3. Table Setup and Partitioning

### 3.1. Glue Data Catalog

- Athena uses the AWS Glue Data Catalog as its default metadata store
- Table schemas define the column names, data types, and S3 location
- Security Lake and CloudTrail can auto-populate the Glue Catalog
- You can also use Athena's built-in DDL (CREATE TABLE, ALTER TABLE)

### 3.2. Partitioning for Performance and Cost

- Partitioning is the **most important optimization** for Athena
- Common partitions: `region`, `year`, `month`, `day`, `account`
- Queries that filter on partition columns scan **only the relevant partitions**
- Without partition filtering, Athena may scan all data — increasing cost and latency

```sql
-- Efficient query (filters on partition)
SELECT *
FROM cloudtrail_logs
WHERE region = 'us-east-1'
  AND year = 2026
  AND month = 5;

-- Inefficient query (scans all partitions)
SELECT *
FROM cloudtrail_logs
WHERE eventname = 'ConsoleLogin';
```

**Exam scenario**: A security team runs CloudTrail queries on Athena, but costs are high → the queries are likely not filtering on partition columns (region, year, month, day). Partition pruning is missing.

### 3.3. How CloudTrail Partitions Data

CloudTrail's S3 prefix structure: `AWSLogs/<account>/CloudTrail/<region>/<year>/<month>/<day>/`

When creating an Athena table for CloudTrail, you typically partition by region, year, month, and day.

## 4. Cost Optimization

**Pricing model**: $5.00 per TB of data scanned (varies by region)

| Optimization                       | How It Reduces Cost                                           |
| ---------------------------------- | ------------------------------------------------------------- |
| **Partitioning**                   | Only scan relevant partitions — reduces data scanned          |
| **Columnar formats (Parquet/ORC)** | Scan only the columns needed, not entire rows                 |
| **Compression**                    | Less data to scan (Snappy, Gzip, etc.)                        |
| **Convert to columnar**            | Convert JSON/CSV logs to Parquet for up to 90% cost reduction |
| **Limit columns in SELECT**        | Avoid `SELECT *` — specify only needed columns                |
| **Use LIMIT**                      | For exploratory queries, use LIMIT to reduce scanned data     |

**Format efficiency**: Parquet (columnar) is dramatically more efficient than JSON or CSV. A query on Parquet may scan 1/10th the data of the same query on JSON.

**Exam scenario**: The security team wants to reduce Athena query costs for CloudTrail analysis → convert the CloudTrail logs from JSON to **Parquet** format and ensure queries filter on **partition columns**.

## 5. Integration with Other Services

| Service           | Integration                                                            |
| ----------------- | ---------------------------------------------------------------------- |
| **CloudTrail**    | Athena is the standard tool for querying CloudTrail logs in S3         |
| **AWS Config**    | Query Config snapshot/history data stored in S3                        |
| **Security Lake** | Athena queries OCSF-normalized Parquet data from Security Lake         |
| **VPC Flow Logs** | Query VPC Flow Logs published to S3                                    |
| **QuickSight**    | Visualize Athena query results in dashboards                           |
| **Glue**          | Data Catalog stores table schemas; Glue Crawlers auto-discover schemas |
| **Lambda**        | Run scheduled Athena queries via Lambda for periodic log analysis      |

### 5.1. CloudTrail + Athena

- CloudTrail logs are delivered to S3 in JSON format (gzipped)
- A CloudTrail-specific Athena table setup is provided by AWS (cloudtrail_console.sql)
- This is the most common security use case for Athena on the exam

### 5.2. Security Lake + Athena

- Security Lake stores data in Parquet format (more efficient than CloudTrail's native JSON)
- Queries run faster and scan less data
- Tables are auto-populated in Glue Data Catalog

## 6. Federated Queries

- Athena supports querying data outside S3 via **federated queries**
- Data sources: CloudWatch Logs, DynamoDB, RDS, Redshift, HBase, DocumentDB, Elasticsearch
- Uses Lambda-based connectors
- Enables queries that join S3 log data with real-time data in other sources

**Exam scenario**: An analyst needs to join CloudTrail logs (in S3) with currently active IAM users (in DynamoDB) → use **Athena federated queries** with a DynamoDB connector.

## 7. Security Best Practices

### 7.1. Access Control

- Use **IAM policies** to control who can run Athena queries
- Restrict which S3 buckets Athena can access (via Athena workgroup settings or S3 bucket policies)
- Use **Athena workgroups** to separate query access, control costs, and enforce query limits
- Output results bucket should have **deny public access** and appropriate encryption

### 7.2. Query Monitoring

- **CloudTrail** logs all Athena API calls (StartQueryExecution, StopQueryExecution, etc.)
- **CloudWatch** metrics are available for query execution
- **Athena workgroup** settings can enforce query cost limits

### 7.3. Encryption

- **S3 encryption** on the log data bucket (SSE-S3 or SSE-KMS)
- **S3 encryption** on the query results bucket
- **KMS** can be used for both

## 8. Limits

| Resource                       | Limit                    |
| ------------------------------ | ------------------------ |
| Max query timeout              | 30 minutes               |
| Max result set size            | 2,500 GB (per query)     |
| Max DML query length           | 262,144 characters       |
| Concurrent queries per account | 20 (default, adjustable) |
| Max partition values per query | 1,000,000                |
| Max table partitions           | 100,000 per table        |

## 9. Exam Tips

1. **Athena = querying S3 logs with SQL**. If the scenario involves running ad-hoc SQL queries on CloudTrail logs, VPC Flow Logs, or Config data stored in S3, the answer is Athena.

2. **Partitioning is critical for cost**. Always partition by region/date. Queries that filter on partitions scan less data and cost less.

3. **Parquet format** reduces cost by up to 90% compared to JSON — columnar format means only queried columns are scanned.

4. **$5 per TB scanned** — cost is based on data scanned, not results returned.

5. **Serverless** — no infrastructure to manage. Just define the schema and query.

6. **Glue Data Catalog** stores table definitions. Glue Crawlers can auto-discover schemas from S3 data.

7. **CloudTrail + Athena** is the standard pattern for log analysis. AWS provides a setup script for creating the CloudTrail table in Athena.

8. **Security Lake + Athena** is more efficient than CloudTrail + Athena because Security Lake uses Parquet while CloudTrail uses JSON.

9. **Federated queries** enable joining S3 data with CloudWatch Logs, RDS, DynamoDB, etc.

10. **Workgroups** separate query access between teams and can enforce cost controls.

11. **Athena is not for real-time** — queries run on data already in S3 (minutes to hours old). For real-time, use CloudWatch Logs Insights or subscription filters.

12. **Avoid SELECT \*** — only query the columns you need to reduce data scanned.

13. **Use LIMIT** for exploratory queries to cap data scanned.

14. **QUERYSTRING** in Athena Workgroups can limit the maximum data scanned per query (e.g., limit to 10TB per query).

15. **Athena vs CloudWatch Logs Insights**: Athena queries S3 data (cost per TB scanned). CloudWatch Logs Insights queries CloudWatch Logs data (cost per GB ingested). Athena is better for large-scale historical analysis; CloudWatch Logs Insights is better for recent data with real-time context.
