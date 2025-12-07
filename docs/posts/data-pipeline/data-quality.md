Below is a **production-ready implementation plan** for Data Quality (DQ) rules within a **medallion architecture** (Bronze/Silver/Gold layers) on **AWS S3**, designed for enterprise-scale structured/unstructured data. I've integrated AWS-native services, governance, and security best practices based on your scenario.

---

### **Core Principles Applied**
1. **Layer-Specific DQ Focus**  
    - *Bronze*: Raw integrity (file-level, schema consistency)  
    - *Silver*: Business logic & cleansing (validity, referential integrity)  
    - *Gold*: Analytics readiness (consistency, aggregation rules)  
2. **Automated & Auditable**  
    - DQ rules stored in metadata catalog (Glue Data Catalog)  
    - Alerts routed to security/governance teams via CloudWatch/SNS  
3. **Security-First**  
    - DQ rules enforce access controls (Lake Formation)  
    - Sensitive data masked in DQ reports (Macie)  

---

### **Implementation: Data Quality Rules by Layer**
#### **1. Bronze Layer (Raw Ingestion)**  
*Goal: Ensure raw data arrives intact, unmodified, and cataloged.*  
| **Rule Type**       | **Rule**                                                                 | **AWS Service**         | **Why It Matters**                                                                 |
|----------------------|--------------------------------------------------------------------------|-------------------------|----------------------------------------------------------------------------------|
| **File Integrity**   | All files must have non-zero size; no empty directories.                  | S3 Event Notifications + Lambda | Prevents broken pipelines from empty source systems.                               |
| **Schema Consistency** | All CSV/JSON files must match defined schema (e.g., `customer_id: string`). | AWS Glue Data Catalog + DQ Rules | Ensures downstream layers don’t fail on unexpected columns.                       |
| **Unstructured Data**| PDFs must have valid MIME type (`application/pdf`); images must be within size limits (e.g., <10MB). | S3 Object Metadata + Lambda | Prevents malicious/invalid files from entering the pipeline.                      |
| **Ingestion Timeliness** | Data must arrive within 15 mins of source event (for streaming) or 1 hour (batch). | Kinesis Data Streams + CloudWatch Alarms | Critical for time-sensitive analytics (e.g., fraud detection).                    |

**Implementation**:  
- **Glue Data Catalog** stores schema definitions (e.g., `bronze.customer_data`).  
- **Lambda** validates file size/MIME type on S3 `PutObject` events.  
- **DQ Rule Definition**:  
  ```json
  {
    "rule_name": "bronze_file_size_check",
    "layer": "bronze",
    "rule_type": "file_size",
    "threshold": 1,
    "action": "fail_pipeline",
    "catalog_entry": "s3://bronze/customer_data"
  }
  ```

---

#### **2. Silver Layer (Cleansed & Validated)**  
*Goal: Enforce business rules, handle nulls, and resolve inconsistencies.*  
| **Rule Type**           | **Rule**                                                                 | **AWS Service**         | **Why It Matters**                                                                 |
|--------------------------|--------------------------------------------------------------------------|-------------------------|----------------------------------------------------------------------------------|
| **Business Validity**    | `order_amount > 0` (for sales data); `customer_age >= 18` (for marketing). | Glue ETL + PySpark DQ Checks | Prevents invalid analytics (e.g., negative revenue).                               |
| **Referential Integrity** | `order.customer_id` must exist in `customer.customer_id` (join validation). | Glue Data Catalog + PySpark | Avoids orphaned records in downstream reports.                                    |
| **Null Handling**        | `email` must be non-null (critical for marketing); `phone` can be null.   | PySpark `.na.drop()` + Custom Rule | Ensures key fields are available for downstream use.                              |
| **Data Type Consistency** | `order_date` must be ISO 8601 format (e.g., `2023-10-05`).               | PySpark `.withColumn` + Regex | Prevents parsing errors in analytics tools (e.g., QuickSight).                    |

**Implementation**:  
- **Glue Data Catalog** stores DQ rules as table properties (e.g., `silver.customer_data.dq_rules`).  
- **PySpark DQ Script** (executed in Glue ETL):  
  ```python
  # Example: Business rule validation
  from pyspark.sql import functions as F

  # Fail if >1% of orders have negative amounts
  if silver_df.filter(F.col("order_amount") < 0).count() / silver_df.count() > 0.01:
      raise ValueError("Invalid order amounts > 1%")
  ```
- **Failure Handling**:  
    - Pipeline stops → S3 `bronze/failed/` bucket (with error details)  
    - CloudWatch Alarm → SNS notification to Data Governance team  

---

#### **3. Gold Layer (Analytics-Ready)**  
*Goal: Ensure data is consistent for business reporting and AI/ML.*  
| **Rule Type**           | **Rule**                                                                 | **AWS Service**         | **Why It Matters**                                                                 |
|--------------------------|--------------------------------------------------------------------------|-------------------------|----------------------------------------------------------------------------------|
| **Aggregation Consistency** | `daily_active_users` must equal sum of `user_sessions` per day.           | Athena + QuickSight DQ | Prevents conflicting metrics in executive dashboards.                             |
| **Data Drift Detection** | `customer_age_avg` must stay within ±5% of historical average (via SageMaker). | SageMaker Model Monitor | Flags data degradation (e.g., new source system injecting invalid ages).          |
| **Sensitive Data Masking** | `SSN` must be masked in all reports (via Lake Formation permissions).     | AWS Lake Formation + Macie | Ensures GDPR/CCPA compliance in DQ reports.                                      |

**Implementation**:  
- **SageMaker Model Monitor** tracks drift in Gold layer metrics (e.g., `revenue` distribution).  
- **Lake Formation** enforces DQ rule visibility:  
  ```sql
  -- Only Data Engineers see raw DQ failures
  CREATE ROLE dq_engineers;
  GRANT SELECT ON TABLE silver.dq_failure_logs TO ROLE dq_engineers;
  ```
- **DQ Report**:  
  ```json
  {
    "gold_sales_daily": {
      "consistency_check": "daily_revenue_matches_sum_sessions",
      "status": "PASS",
      "threshold": "±0.1%",
      "actual_diff": "0.03%"
    }
  }
  ```

---

### **Governance & Security Integration**
| **Component**               | **Implementation**                                                                 | **Governance Benefit**                                                                 |
|-----------------------------|----------------------------------------------------------------------------------|--------------------------------------------------------------------------------------|
| **Metadata Catalog**        | All DQ rules stored in **Glue Data Catalog** (as table properties).              | Auditable trail for compliance (e.g., SOC 2, ISO 27001).                             |
| **Access Control**          | Lake Formation policies restrict DQ rule access: `Data Engineers` → full access; `Analysts` → read-only. | Prevents tampering with DQ rules (security control).                                |
| **Alerting**                | CloudWatch alarms → SNS → Slack/email to **Data Governance Team** (with AWS IAM roles). | Ensures rapid response to DQ failures (SLA: <15 mins).                               |
| **Audit Logs**              | All DQ rule executions logged to **CloudTrail** + **S3 Access Logs**.             | Meets forensic requirements for data incidents.                                      |

---

### **Why This Works for Your Scenario**
- **Scalable**: Uses serverless AWS services (Lambda, Glue, Athena) – no infrastructure management.  
- **Enterprise-Ready**:  
    - Handles **structured** (SQL tables, CSV) and **unstructured** (PDFs, images) data.  
    - Integrates with **AI/ML** (SageMaker drift detection) and **analytics** (QuickSight).  
- **Compliance**:  
    - DQ rules enforce **data masking** (Macie) and **access control** (Lake Formation).  
    - Full audit trail for governance teams.  
- **Cost-Effective**:  
    - DQ checks run *only* on processed data (not raw files), avoiding S3 storage costs.  

---

### **Critical Next Steps for Deployment**
1. **Define DQ Rules Catalog**:  
   ```bash
   aws glue update-table --database-name enterprise_data --table-name bronze.customer_data \
   --table-input '{"StorageDescriptor": {"SerDeInfo": {"SerializationLibrary": "org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe"}}}'
   ```
2. **Deploy Glue DQ Job**:  
    - Use **Glue Data Quality** (AWS-native) to run rules against Silver/Gold tables.  
3. **Integrate with Governance**:  
    - Map DQ rules to **AWS Control Tower** guardrails for automated compliance checks.  

> 💡 **Pro Tip**: Start with **Bronze layer DQ** (file integrity) – 80% of pipeline failures are due to missing/invalid raw files. *This delivers 90% of DQ value with minimal effort.*

---

This implementation ensures **zero data quality debt** while meeting enterprise requirements for governance, security, and AI readiness. All rules are automated, auditable, and enforced at the layer where they matter most – **no manual checks, no ad-hoc fixes**.