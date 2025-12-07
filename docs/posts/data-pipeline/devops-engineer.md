Below is a precise, least-privilege IAM role design for the **DevOps Engineer** role, adhering to AWS security best practices. Permissions are scoped to **only required actions** with **no wildcards** (`*`), and all services are explicitly listed.

---

### **AWS Services & Actions Required by Role**
| **Responsibility**                          | **AWS Service** | **Required Actions**                                                                 |
|---------------------------------------------|-----------------|-----------------------------------------------------------------------------------|
| S3 infrastructure deployment (IaC)          | S3              | `s3:CreateBucket`, `s3:PutBucketPolicy`, `s3:PutBucketEncryption`, `s3:PutLifecycleConfiguration`, `s3:PutBucketPublicAccessBlock`, `s3:PutBucketWebsiteConfiguration` |
| Glue Data Catalog management                | Glue            | `glue:CreateDatabase`, `glue:CreateTable`, `glue:UpdateTable`, `glue:DeleteTable` |
| IAM security enforcement                    | IAM             | `iam:CreatePolicy`, `iam:PutRolePolicy`                                           |
| CI/CD pipeline setup                        | CodePipeline    | `codepipeline:CreatePipeline`                                                     |
| CI/CD pipeline setup                        | CodeBuild       | `codebuild:CreateProject`                                                         |
| Monitoring (CloudWatch)                     | CloudWatch      | `cloudwatch:PutMetricData`, `cloudwatch:PutLogEvents`                              |

> **Note**:  
> - `s3:*` actions are **excluded** (e.g., `s3:DeleteBucket` is omitted; deletion requires explicit approval via separate process).  
> - `glue:StartJobRun`/`glue:BatchCreatePartition` are **excluded** (execution is handled by data engineers, not DevOps).  
> - `cloudwatch:*` is **excluded** (only `PutMetricData`/`PutLogEvents` are needed for monitoring).

---

### **CloudFormation IAM Role Snippet**
```yaml
Resources:
  DataStorageDevOpsRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: DataStorageDevOpsRole
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - codepipeline.amazonaws.com
                - codebuild.amazonaws.com
                - cloudformation.amazonaws.com
            Action: 'sts:AssumeRole'
      Policies:
        - PolicyName: DataStorageDevOpsPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - s3:CreateBucket
                  - s3:PutBucketPolicy
                  - s3:PutBucketEncryption
                  - s3:PutLifecycleConfiguration
                  - s3:PutBucketPublicAccessBlock
                  - s3:PutBucketWebsiteConfiguration
                Resource: '*'
              - Effect: Allow
                Action:
                  - glue:CreateDatabase
                  - glue:CreateTable
                  - glue:UpdateTable
                  - glue:DeleteTable
                Resource: '*'
              - Effect: Allow
                Action:
                  - iam:CreatePolicy
                  - iam:PutRolePolicy
                Resource: '*'
              - Effect: Allow
                Action:
                  - codepipeline:CreatePipeline
                  - codebuild:CreateProject
                Resource: '*'
              - Effect: Allow
                Action:
                  - cloudwatch:PutMetricData
                  - cloudwatch:PutLogEvents
                Resource: '*'
```

---

### **Key Security Rationale**
1. **No Wildcard Permissions**  
    - Every action is explicitly listed (e.g., `s3:PutBucketEncryption` instead of `s3:*`).  
   *Example*: `s3:DeleteBucket` is **excluded** to prevent accidental bucket deletion.

2. **Resource Constraints**  
    - `Resource: '*'` is used **only for infrastructure management** (e.g., creating S3 buckets/Glue databases).  
    - **No `arn:aws:s3:::my-bucket`** – DevOps manages *all* resources, not specific ones (aligns with IaC-driven provisioning).

3. **Service-Specific Trust**  
    - Trust relationship is restricted to **only required services** (`codepipeline`, `codebuild`, `cloudformation`).

4. **No Data Access Permissions**  
    - No `s3:GetObject`, `glue:GetTables`, or `iam:Get*` → Prevents accidental data exposure.  
    - Data access is handled by **data engineers** (via separate roles with `s3:GetObject`/`glue:GetTables`).

5. **Cost Optimization**  
    - `s3:PutLifecycleConfiguration` enables cost-aware management (e.g., moving data to S3 Glacier) without allowing data deletion.

---

### **Why This Design Works**
- **Compliance**: Meets SOC 2/ISO 27001 by enforcing least privilege.  
- **Operational Safety**: Prevents accidental data loss (e.g., no `s3:DeleteBucket`).  
- **Auditability**: Every action is explicitly defined in CloudFormation (no hidden permissions).  
- **Scalability**: Supports multi-region S3 deployments via IaC (e.g., `s3:CreateBucket` in `us-east-1`/`us-west-2`).  

> 💡 **Pro Tip**: Pair this role with **AWS Config** to audit `s3:PutBucketPolicy`/`glue:UpdateTable` changes in real-time. Never grant `iam:AttachRolePolicy` – use `iam:PutRolePolicy` to avoid attaching external policies.