
-----

# Project: Secure CI/CD Pipeline for SilvanX.com

This project documents the implementation of a fully automated and securely authenticated **Continuous Deployment (CD) pipeline** for the SilvanX.com static website, hosted on **Amazon S3** and delivered via **Amazon CloudFront**. This repository serves as a guide to solving complex security and configuration conflicts.

-----

## I. Goals Achieved & Final Architecture

| Architecture | Goal | Status | Why |
| :--- | :--- | :--- | :--- |
| **Authentication** | Eliminate storing long-term credentials (Access Keys). | **ACHIEVED** | **IAM OIDC Federation** (Keyless Authentication) |
| **S3 Security** | Keep the S3 bucket private while allowing CloudFront access. | **ACHIEVED** | **OAC** Principles (Securing the Origin) |
| **Functionality** | Enable seamless updates via `git push` that resolve complex pathing issues. | **ACHIEVED** | Automated Invalidation & Origin Pathing |

-----

## II. Key Configuration Files

### A. CI/CD Deployment Workflow (`.github/workflows/deploy.yml`)

This YAML file executes the deployment securely by assuming an IAM role via OIDC and using the correct file ownership flags.

```yaml
name: Deploy SilvanX Static Website
on:
  push:
    branches:
      - main
permissions:
  id-token: write # Enables OIDC Token Generation
  contents: read
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4
      
      - name: Configure AWS Credentials using OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          # IAM Role assumed via OIDC.
          role-to-assume: arn:aws:iam::AWS_ACCOUNT_ID:role/SilvanX-GitHub-Deployment-Role 
          aws-region: us-east-1 

      - name: Deploy files to S3 (Sync & Set Ownership)
        # --acl bucket-owner-full-control is critical to prevent "Access Denied" on new objects
        run: aws s3 sync . s3://S3_BUCKET_NAME --acl bucket-owner-full-control --delete

      - name: Invalidate CloudFront Cache
        run: aws cloudfront create-invalidation --distribution-id CF_DISTRIBUTION_ID --paths "/*"
```

### B. Final Working S3 Bucket Policy (Multiple Principal Access)

This policy was the necessary final combination required to resolve a permission conflicts in the CloudFront/S3 configuration.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        // 1. PUBLIC READ (Required to satisfy an unknown account-level restriction or endpoint conflict)
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::S3_BUCKET_NAME/*"
        },
        // 2. LEGACY OAI ACCESS (Included for compatibility with linked CloudFront OAI identity)
        {
            "Sid": "2",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity OAI_ID"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::S3_BUCKET_NAME/*"
        },
        // 3. MODERN OAC PRINCIPAL (The preferred, secure method)
        {
            "Sid": "AllowCloudFrontServicePrincipal1",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudfront.amazonaws.com"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::S3_BUCKET_NAME/*",
            "Condition": {
                "StringEquals": {
                    "AWS:SourceArn": "arn:aws:cloudfront::AWS_ACCOUNT_ID:distribution/CF_DISTRIBUTION_ID"
                }
            }
        }
    ]
}
```

-----

## III. Problems Encountered and Solutions

| Problem Encountered | Root Cause & Security Implication | Final Solution |
| :--- | :--- | :--- |
| **Credential Exposure** | Initial plan required static Access Keys. | Implemented **IAM OIDC Federation** for keyless, temporary access. |
| **Object Read Denial** | Deployment role failed to grant object ownership back to the bucket owner. | Added **`--acl bucket-owner-full-control`** to the `aws s3 sync` command. |
| **Endpoint Conflict** | S3 bucket had **Static Website Hosting** enabled, conflicting with the secure **REST API Endpoint**. | Disabled **S3 Static Website Hosting** on the bucket. |
| **Root Access Failure** | The website content was nested in `/SilvanX_Website/`, causing the visitor's request (`/`) to fail. | Set the **CloudFront Origin Path** to **`/SilvanX_Website`** to resolve the final path mismatch. |

-----

## IV. Conclusion

The single greatest challenge of this project was diagnosing and fixing the persistent **`AccessDenied`** XML error at the website root (`https://www.silvanx.com/`) immediately after implementing the secure OAC/OAI access controls.

### The Core Conflict

The root cause was a conflict between three different S3 access mechanisms: Public Read, Legacy OAI, and Modern OAC. Each configuration change resulted in a temporary fix that was subsequently blocked by an overriding setting, ultimately revealed to be an **endpoint path mismatch**.

### Problem-Solving Steps & Takeaways

| Step | Hypothesis Tested | Status | Key Skill Demonstrated |
| :--- | :--- | :--- | :--- |
| **1. File Ownership/ACLs** | Files uploaded by the IAM Role were owned by the Role, blocking the S3 Bucket Owner's access. | **Fixed** by adding **`--acl bucket-owner-full-control`** to the `aws s3 sync` command. | Least Privilege, Object Ownership. |
| **2. Bucket Policy** | The secure OAC/OAI policy was incorrectly formatted, rejected by the S3 console. | **Fixed** by using the final, cumulative policy that satisfied the strict Principal checks of S3/IAM. | Policy Synthesis & Validation. |
| **3. Static Website Endpoint** | The website was returning an XML error because the **S3 Website Hosting** feature conflicted with the secure **OAC/REST API Endpoint**. | **Fixed** by **Disabling Static Website Hosting** in S3 bucket properties. | SAA Exam Knowledge (Endpoint Incompatibility). |
| **4. Final Path Mismatch** | The visitor request (`/`) was being sent to S3 as the root key, which did not exist because files were in a subfolder (`/SilvanX_Website`). | **Fixed** by setting the **CloudFront Origin Path** to **`/SilvanX_Website`**. | Deep CloudFront Behavior & Request Mapping. |

Thank You - Cat Silvan

[SilvanX-Website Source Repository](https://github.com/CatrickLee/SilvanX-Website)

[View the original deploy.yml file](https://github.com/CatrickLee/SilvanX-Website/blob/main/.github/workflows/deploy.yml)

**[SilvanX.com (Live Site)](https://www.silvanx.com/)**
