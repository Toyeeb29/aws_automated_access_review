[![Tests](https://github.com/ajy0127/aws_automated_access_review/actions/workflows/tests.yml/badge.svg)](https://github.com/ajy0127/aws_automated_access_review/actions/workflows/tests.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![CFN Lint](https://img.shields.io/badge/CFN-Lint-blue.svg)](https://github.com/aws-cloudformation/cfn-lint)
[![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/)
[![Code Style: Black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)

# 🔐 AWS Automated Access Review

A serverless solution for automating AWS IAM access reviews using Lambda, Amazon Bedrock (Claude AI), Security Hub, and IAM Access Analyzer. Automatically generates access review reports and delivers them via email on a scheduled basis.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Tech Stack](#tech-stack)
- [Setup & Installation](#setup--installation)
- [Deployment](#deployment)
- [Running a Report](#running-a-report)
- [AWS Resources Deployed](#aws-resources-deployed)
- [Sample Report Output](#sample-report-output)
- [Troubleshooting](#troubleshooting)
- [Known Issues](#known-issues)

---

## Overview

This project automates the process of reviewing AWS IAM access across your environment. It runs on a **30-day schedule** and:

- Scans IAM users, roles, and policies via **IAM Access Analyzer**
- Surfaces security findings from **AWS Security Hub**
- Generates AI-powered summaries using **Amazon Bedrock (Claude Haiku)**
- Saves reports as CSV to **Amazon S3**
- Delivers findings to a configured email via **Amazon SES**

---

## Architecture

```
CloudWatch Events (30-day schedule)
        │
        ▼
  Lambda Function  ──────────────────────────────────────────┐
  (Python 3.11)                                              │
        │                                                     │
        ├──► IAM Access Analyzer  (access findings)          │
        ├──► AWS Security Hub     (security findings)        │
        ├──► Amazon Bedrock       (AI-powered analysis)      │
        ├──► Amazon S3            (report storage as CSV)    │
        └──► Amazon SES           (email delivery)           │
                                                             │
  CloudFormation Stack manages all resources ◄──────────────┘
```

---

## Prerequisites

- **AWS Account** with the following services enabled:
  - AWS Security Hub
  - IAM Access Analyzer
  - Amazon SES (with a verified email address)
  - Amazon Bedrock (with Claude Haiku model access)
- **AWS CLI** configured with valid credentials
- **Python 3.11+**
- **Git Bash** (Windows) or a Unix shell
- **zip** utility (see [Troubleshooting](#troubleshooting) for Windows notes)

---

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Python 3.11 (AWS Lambda) |
| Infrastructure | AWS CloudFormation |
| AI Analysis | Amazon Bedrock — Claude Haiku |
| Access Analysis | IAM Access Analyzer |
| Security Findings | AWS Security Hub |
| Report Storage | Amazon S3 |
| Email Delivery | Amazon SES |
| Scheduling | Amazon CloudWatch Events |

---

## Setup & Installation

### 1. Clone the Repository

```bash
git clone https://github.com/Toyeeb29/aws_automated_access_review.git
cd aws_automated_access_review
```

### 2. Create and Activate a Virtual Environment

```bash
python -m venv venv

# Windows (Git Bash)
source venv/Scripts/activate

# macOS / Linux
source venv/bin/activate
```

### 3. Install Dependencies

```bash
pip install -r requirements.txt
```

### 4. Verify AWS Credentials

```bash
./scripts/check_aws_creds.sh
```

Expected output confirms access to:
- ✅ Security Hub
- ✅ IAM Access Analyzer
- ✅ Amazon SES
- ✅ Amazon Bedrock

![Valid AWS Credentials Output]

(<img width="384" height="171" alt="Valid Credentials Output" src="https://github.com/user-attachments/assets/0b8796c1-dff3-4eea-ab31-fef557d3b0b5" />
)

### 5. Verify Your Email with SES

Before deploying, make sure your email is registered with SES:

```bash
aws ses list-identities
```

If your email is not listed, verify it:

```bash
aws ses verify-email-identity --email-address your.email@example.com
```

Then check your inbox for a verification link from AWS.

---

## Deployment

```bash
./scripts/deploy.sh --email your.email@example.com
```

The script will:
1. Validate your AWS credentials
2. Package the Lambda function
3. Deploy the CloudFormation stack (`aws-access-review`)
4. Upload the Lambda code to S3
5. Output the deployed resource ARNs

### Verify Stack Status

```bash
aws cloudformation describe-stacks \
  --stack-name aws-access-review \
  --query 'Stacks[0].StackStatus'
```

Expected: `"UPDATE_COMPLETE"` or `"CREATE_COMPLETE"`

![Deployment Complete Output]

(<img width="592" height="176" alt="deployment complete output" src="https://github.com/user-attachments/assets/5e2f2ca2-30f4-44db-a8a3-6d32a126e62d" />
)!

---

## Running a Report

### Via the Helper Script

```bash
./scripts/run_report.sh
```

### Via AWS CLI (direct Lambda invocation)

```bash
aws lambda invoke \
  --function-name aws-access-review-access-review \
  --payload '{}' response.json

cat response.json
```

A successful run returns:

```json
{
  "statusCode": 200,
  "body": "\"AWS Access Review completed successfully\"",
  "reportDetails": {
    "timestamp": "2026-04-21-16-31-43",
    "bucket": "aws-access-review-reportbucket-XXXXXXXX",
    "key": "reports/aws-access-review-2026-04-21-16-31-43.csv",
    "findingsCount": 11
  }
}
```

Reports are saved to S3 and emailed to the configured recipient automatically.

---

## AWS Resources Deployed

| Resource | Type |
|---|---|
| `AccessReviewLambda` | AWS::Lambda::Function |
| `AccessReviewLambdaRole` | AWS::IAM::Role |
| `ReportBucket` | AWS::S3::Bucket |
| `ReportBucketPolicy` | AWS::S3::BucketPolicy |
| `ScheduledRule` | AWS::Events::Rule |
| `PermissionForEventsToInvokeLambda` | AWS::Lambda::Permission |

---

## Sample Report Output

The following is an example AI-generated security assessment report produced by this tool from a real lab deployment. The report was generated by Amazon Bedrock (Claude Haiku) based on 11 findings surfaced by Security Hub and IAM Access Analyzer.

---

### Executive Summary

> **11 security findings** identified — **2 high-priority issues** requiring immediate remediation. No critical vulnerabilities found, but the combination of disabled logging, unprotected user access, and outdated credentials presents significant security and compliance risks.

**Overall Security Posture: ⚠️ MODERATE RISK**

---

### 🔴 High Priority Findings

#### 1. CloudTrail Logging Disabled
**Impact:** Complete loss of audit trail and compliance violation

- CloudTrail is not enabled — no API activity is being logged or monitored
- Inability to detect unauthorized access, investigate incidents, or meet regulatory requirements (SOC 2, PCI-DSS, HIPAA)

**Remediation:**
- Enable CloudTrail immediately for all regions
- Configure S3 bucket with encryption and versioning for log storage
- Set up CloudWatch alarms for critical API calls
- **Target:** Complete within 48 hours

#### 2. Unprotected Console Access (No MFA)
**Impact:** Compromised credentials could grant unrestricted AWS access

- User `test-console-user` has Console access without Multi-Factor Authentication
- Password compromise = full account access with no additional verification

**Remediation:**
- Enforce MFA immediately for this user
- Consider AWS SSO with org-wide MFA enforcement
- Remove console access from service accounts
- **Target:** Complete within 24 hours

---

### 🟡 Medium Priority Findings

| Issue | Details | Action |
|-------|---------|--------|
| **Stale Access Keys** | 3 users have keys 100–105 days old | Rotate all keys older than 90 days |
| **Excessive Privileges** | User `fikayo` has `AdministratorAccess` | Apply least-privilege; use role-based access |
| **External Access** | Unidentified external entity has resource access | Review Access Analyzer findings; remove unintended permissions |
| **No SCPs** | Organization lacks guardrails to prevent risky actions | Implement Service Control Policies |

---

### ✅ Recommended Action Plan

**Immediate (24–48 hours)**
- [ ] Enable CloudTrail across all regions
- [ ] Enforce MFA for `test-console-user`
- [ ] Rotate all IAM access keys older than 90 days

**Short-term (1–2 weeks)**
- [ ] Review and reduce `fikayo` user privileges
- [ ] Investigate external access flagged by Access Analyzer
- [ ] Implement org-wide MFA requirement

**Medium-term (30 days)**
- [ ] Deploy Service Control Policies (SCPs)
- [ ] Establish automated credential rotation
- [ ] Configure Security Hub for continuous monitoring

---

### Compliance Implications

**Current Status: ❌ NON-COMPLIANT** with common frameworks

| Framework | Impact |
|-----------|--------|
| **SOC 2** | CloudTrail required for audit logging |
| **PCI-DSS** | MFA required for administrative access |
| **HIPAA** | Comprehensive audit logging mandatory |
| **CIS AWS Foundations** | Multiple control failures |

Remediating the two high-priority findings will restore compliance readiness across all frameworks above.

---

### Severity Distribution

| Severity | Count |
|----------|-------|
| 🔴 High | 2 |
| 🟡 Medium | 8 |
| 🔵 Informational | 1 |
| **Total** | **11** |

> The full findings are available as a CSV file stored in the S3 report bucket generated during deployment.

[📄 Download Sample CSV Report](assets/aws-access-review-2026-04-21-16-31-43.csv)

### Raw Findings (CSV)

| ID | Category | Severity | Resource Type | Resource ID | Description | Recommendation | Compliance |
|----|----------|----------|---------------|-------------|-------------|----------------|------------|
| IAM-002-AKIAWWXFC3ZSC6YCF6EA | IAM | Medium | IAM Access Key | fikayo/AKIAWWXFC3ZSC6YCF6EA | Access key is 105 days old | Rotate keys every 90 days | CIS 1.4, AWS Well-Architected |
| IAM-002-AKIAWWXFC3ZSEELS7QFE | IAM | Medium | IAM Access Key | fikayo/AKIAWWXFC3ZSEELS7QFE | Access key is 105 days old | Rotate keys every 90 days | CIS 1.4, AWS Well-Architected |
| IAM-003-fikayo | IAM | Medium | IAM User | fikayo | User has wide privileges via AdministratorAccess | Apply least privilege principle | CIS 1.16, AWS Well-Architected |
| IAM-002-AKIAWWXFC3ZSJUKAKK5K | IAM | Medium | IAM Access Key | Imran/AKIAWWXFC3ZSJUKAKK5K | Access key is 100 days old | Rotate keys every 90 days | CIS 1.4, AWS Well-Architected |
| IAM-003-Imran | IAM | Medium | IAM User | Imran | User has wide privileges via AdministratorAccess | Apply least privilege principle | CIS 1.16, AWS Well-Architected |
| IAM-001-test-console-user | IAM | **High** | IAM User | test-console-user | Console access without MFA | Enable MFA for all console users | CIS 1.2, AWS Well-Architected |
| IAM-005 | IAM | Medium | IAM Password Policy | account-password-policy | Password policy does not meet best practices | Require 14+ characters with mixed types | CIS 1.5–1.11, AWS Well-Architected |
| SCP-001 | SCP | Medium | Service Control Policy | none | No custom SCPs detected | Implement SCPs for security guardrails | AWS Well-Architected |
| SECHUB-POSITIVE-001 | SecurityHub | Informational | AWS Security Hub | none | No high/critical IAM findings detected | Continue monitoring | AWS Well-Architected |
| AA-7b108fad | Access Analyzer | Medium | Unknown | Unknown | External access may not be intended | Review and restrict permissions | AWS Well-Architected, CIS AWS Foundations |
| CT-NOT-ENABLED | CloudTrail | **High** | AWS CloudTrail | none | CloudTrail is not enabled | Enable CloudTrail to track API activity | AWS Well-Architected |

**Detection Date:** 2026-04-21 · **Total Findings:** 11 · **High:** 2 · **Medium:** 8 · **Informational:** 1

---

## Troubleshooting

### `zip.exe` Error on Windows (Git Bash)

If the deploy script fails with a `zip` shared library error, use the Chocolatey version:

```bash
export PATH="/c/ProgramData/chocolatey/bin:$PATH"
echo "alias zip='/c/ProgramData/chocolatey/bin/zip'" >> ~/.bashrc
source ~/.bashrc
```

### SSO Token Expired

If using an AWS SSO profile and credentials are expired:

```bash
aws sso login --profile YOUR_PROFILE
```

Or switch to an IAM user profile in `~/.aws/credentials`.

### SES Email Not Verified

```bash
aws ses verify-email-identity --email-address your.email@example.com
```

Check your inbox for the AWS verification link before deploying.

### Checking Lambda Logs

```bash
aws logs tail /aws/lambda/aws-access-review-access-review --follow
```

Or view in the AWS Console via CloudWatch Logs.

---

## Known Issues

- **Git Bash on Windows:** The `zip` binary bundled with Git Bash may fail due to missing shared libraries. Use the Chocolatey `zip` binary as a workaround (see above).
- **SSO Profiles:** Named AWS SSO profiles may require re-authentication. Default IAM credentials work reliably with the credential check script.
- **First Deployment:** SES email verification must be completed before the Lambda function can deliver reports.

---

## License

This project is intended for educational and GRC (Governance, Risk & Compliance) lab purposes.Chocolatey `zip` binary as a workaround (see above).
- **SSO Profiles:** Named AWS SSO profiles may require re-authentication. Default IAM credentials work reliably with the credential check script.
- **First Deployment:** SES email verification must be completed before the Lambda function can deliver reports.

---

## License

This project is intended for educational and GRC (Governance, Risk & Compliance) engineering lab purposes.
