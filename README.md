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
| Testing | pytest, moto, pytest-mock |
| Linting | flake8, black |
| IaC Linting | cfn-lint |

---

## Setup & Installation

### 1. Clone the Repository

```bash
git clone https://github.com/YOUR_USERNAME/aws_automated_access_review.git
cd aws_automated_access_review
```

> **Note:** If forking from the upstream repo, add it as a remote:
> ```bash
> git remote add upstream https://github.com/ajy0127/aws_automated_access_review.git
> git fetch upstream
> git merge upstream/main
> ```

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

This project is intended for educational and GRC (Governance, Risk & Compliance) lab purposes.
