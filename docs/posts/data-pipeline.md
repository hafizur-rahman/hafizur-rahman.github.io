## Scenario
An enterprise needs a centralized repository for structured and unstructured data to support advanced analytics and AI.

## Architect's Role
* Choose the right storage (AWS S3, Azure Data Lake).
* Define ingestion pipelines (batch vs. streaming).
* Implement metadata management and data cataloging.
* Ensure governance and access control.

## Data Steward
Here's a precise, implementation-focused governance and access control strategy for the **Data Steward** role, aligned with AWS/Azure best practices and regulatory requirements. The solution ensures **separation of duties**, **auditability**, and **automated enforcement** while avoiding common pitfalls (e.g., giving stewards direct data access).


### **Core Principles Applied**
1. **Least Privilege Access**: Stewards *only* get permissions needed for cataloging, not data storage or pipelines.
2. **Automated Enforcement**: Governance policies are embedded in tools, not manual processes.
3. **Separation of Duties**: Stewards define *what* data needs protection; Security teams implement *how*.
4. **Immutable Audit Trails**: All actions are logged for compliance (GDPR/CCPA).

---

### **Implementation Blueprint**

#### **1. Data Catalog & Metadata Management (Enforcing Catalog Entries)**
| **Tool**                     | **Implementation**                                                                 | **Access Control**                                                                 |
|------------------------------|----------------------------------------------------------------------------------|----------------------------------------------------------------------------------|
| **AWS**: Glue Data Catalog + AWS Lake Formation | Stewards use **Lake Formation's Data Catalog** (not S3 directly). All catalog entries (tables, columns, lineage) are managed here. | - **IAM Role**: `LakeFormationDataCatalogAdmin` (limited to `datacatalog:*` actions)<br>- **No S3 bucket permissions** (prevents direct data access)<br>- **Policy**: `Allow lakeformation:PutResource` only for catalog metadata |
| **Azure**: Purview + Azure Data Catalog | Stewards use **Purview's Catalog** to define assets, business terms, and lineage. | - **Azure RBAC**: `Purview Catalog Reader` (with `Catalog Administrator` role *only* for catalog metadata)<br>- **No storage account permissions** (e.g., no `Storage Blob Data Contributor` on ADLS) |

> ✅ **Why this works**: Catalog entries are *metadata*, not data. Stewards cannot access raw data or alter storage permissions.

---

#### **2. Data Quality Standards & Lineage (Enforcing Metrics & Documentation)**
| **Tool**                     | **Implementation**                                                                 | **Access Control**                                                                 |
|------------------------------|----------------------------------------------------------------------------------|----------------------------------------------------------------------------------|
| **AWS**: Glue Data Quality + AWS Audit Manager | Stewards define **data quality rules** (e.g., "nulls ≤ 0.1% in `customer_email`") in Glue Data Catalog. **Lineage** is auto-captured via Glue Data Catalog. | - **IAM Policy**: `Allow glue:CreateDataQualityRuleset` (only for catalog)<br>- **No pipeline permissions** (rules are stored in catalog, not pipelines) |
| **Azure**: Azure Data Factory + Purview | Stewards define **quality rules** in Purview (e.g., "column `SSN` must be encrypted"). **Lineage** is auto-logged in Purview. | - **Purview Role**: `Purview Data Curator` (can manage rules but *not* data storage)<br>- **No Data Factory pipeline permissions** |

> ✅ **Why this works**: Quality rules are *metadata* stored in the catalog. Stewards collaborate with engineers to *apply* rules to pipelines (via API), but **never deploy pipelines themselves**.

---

#### **3. Access Control & Encryption (Collaborating with Security Teams)**
| **Requirement**               | **Implementation**                                                                 | **Governance Enforcement**                                                                 |
|-------------------------------|----------------------------------------------------------------------------------|------------------------------------------------------------------------------------------|
| **Define access policies**    | Stewards **tag data assets** in catalog (e.g., `sensitivity=GDPR`, `owner=finance`). | - **Automated Policy**: Lake Formation/Purview **enforces access** based on tags.<br>Example: `sensitivity=GDPR` → auto-encrypts data + restricts access to GDPR-compliant roles. |
| **Audit access requests**     | All catalog changes (e.g., adding a `GDPR` tag) are logged in **CloudTrail (AWS) / Azure Monitor (Azure)**. | - **Stewards cannot approve access**: Security team uses **Lake Formation Data Permissions** or **Azure RBAC** to grant access *based on catalog tags*. |
| **Compliance (GDPR/CCPA)**    | Purview/Lake Formation **auto-classifies** data using **sensitivity labels** (e.g., `PII`, `PHI`). | - **Stewards document**: `business_definition` and `usage_policy` in catalog.<br>- **Automated checks**: Purview scans for untagged data; flags for remediation. |

> ⚠️ **Critical Avoidance**: Stewards **never** manage IAM roles or storage permissions. Example:  
> ❌ *Steward tries to grant `s3:GetObject` to a user* → **Blocked by IAM policy**.  
> ✅ *Steward adds `sensitivity=CCPA` tag → System auto-applies encryption + restricts access*.

---

#### **4. Data Quality Issue Resolution (Collaboration Workflow)**
| **Step**                      | **Tool**                         | **Access Control**                                                                 |
|-------------------------------|----------------------------------|----------------------------------------------------------------------------------|
| 1. Steward flags quality issue in catalog | Purview/Glue Data Catalog (via UI/API) | - **Only catalog write access** (no pipeline/data access) |
| 2. System auto-creates **Jira ticket** for Data Engineering | AWS Service Catalog / Azure DevOps | - **Data Engineers** get ticket (no steward access) |
| 3. Engineers fix pipeline → **Automated test runs** (Glue Data Quality) | Glue Data Quality | - **Steward verifies** fix via catalog UI (no code access) |

> ✅ **Why this works**: Separates *discovery* (steward) from *fixing* (engineers). No security risk.

---

### **Why This Satisfies All Responsibilities**
| **Steward Responsibility**                                  | **How Solution Enforces It**                                                                 |
|------------------------------------------------------------|-------------------------------------------------------------------------------------------|
| Establish/maintain catalog entries                         | Catalog is *only* managed via Lake Formation/Purview (no direct data access)               |
| Define data quality standards                              | Rules stored in catalog → enforced by pipelines (not stewards)                            |
| Collaborate on access controls                             | Tags in catalog drive *automated* access policies (security team configures rules)        |
| Ensure GDPR/CCPA compliance                                | Auto-classification + catalog documentation → audit-ready evidence                         |
| Document metadata/lineage                                  | Lineage auto-captured in catalog; business definitions stored as metadata                 |
| Resolve data quality issues                                | Workflow via catalog → tickets to engineers (no direct data access)                       |

---

### **Critical Security Safeguards**
1. **No Data Storage Permissions**: Stewards have **zero** access to S3/ADLS buckets (prevents data exfiltration).
2. **Immutable Audit Logs**: All catalog changes logged in CloudTrail/Azure Monitor → **GDPR Article 30 compliance**.
3. **Automated Policy Enforcement**: 
   - *Example (AWS)*: If a table lacks `sensitivity` tag → **Lake Formation blocks access** until tagged.
   - *Example (Azure)*: Purview flags unclassified data → **Security team receives alert**.
4. **Role Separation**: 
   - **Stewards**: `Catalog Administrator` (only catalog)
   - **Security Team**: `Lake Formation Data Permissions Admin` (configures access)
   - **Engineers**: `Glue Data Pipeline Developer` (builds pipelines)

---

### **Why Not Other Approaches?**
- ❌ **Giving stewards S3 permissions**: Violates least privilege → risk of data leaks.
- ❌ **Manual access approvals**: Inefficient, error-prone, non-auditable for GDPR.
- ❌ **Stewards defining IAM policies**: Creates security risks (e.g., over-permissive roles).

---

### **Final Architecture Summary**
```mermaid
graph LR
  A[Data Steward] -->|Tags data in catalog| B(Purview/Lake Formation)
  B -->|Auto-applies policies| C[Security Team: Lake Formation/ADLS Permissions]
  B -->|Auto-classifies data| D[Compliance (GDPR/CCPA)]
  B -->|Creates quality rules| E[Data Engineering: Glue Data Quality]
  E -->|Fixes pipeline| F[Steward verifies via catalog]
```
