<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [AWS Lambda × S3: Secure Architecture for Cross-Account Serverless Image Processing](#aws-lambda-%C3%97-s3-secure-architecture-for-cross-account-serverless-image-processing)
  - [📘 Scenario](#-scenario)
  - [🎯 Test Points](#-test-points)
  - [✅ Summary](#-summary)
  - [🛠️ Implementation Example](#-implementation-example)
    - [In Account B: Create IAM Role (ThumbXRole)](#in-account-b-create-iam-role-thumbxrole)
    - [In Account A: Add AssumeRole permission to Lambda role](#in-account-a-add-assumerole-permission-to-lambda-role)
    - [Lambda Code: AssumeRole Execution](#lambda-code-assumerole-execution)
  - [🔐 Security Checklist](#-security-checklist)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


# AWS Lambda × S3: Secure Architecture for Cross-Account Serverless Image Processing
---

## 📘 Scenario

```
┌───────────┐            ┌──────────────┐
│ Account A │            │  Account B   │
│ (Producer)│            │ (Storage)    │
│           │ put object │              │
│Lambda Fn  ├──────────► │ S3 Bucket    │
│           │ get object │  (Images)    │
└───────────┘            └──────────────┘
```

To safely access an S3 bucket in another AWS account, the Lambda function must be configured with proper authorization using **IAM AssumeRole** or **bucket policies**.  
In this scenario, a Lambda function in Account A generates thumbnails and stores them in an S3 bucket in Account B.  
This document outlines best practices for secure cross-account architecture.

---

## 🎯 Test Points

- How to configure cross-account access permissions
- How to distinguish between IAM roles and trust policies
- Use of `sts:AssumeRole` for temporary credentials
- Use of `Principal` in bucket policy settings
- Security best practices and audit mechanisms

---

## ✅ Summary

| Action | Reason |
|--------|--------|
| Create an IAM role (ThumbXRole) in Account B | Explicitly define permissions |
| Set a trust policy to allow Account A’s Lambda role | Establish cross-account trust |
| Grant S3 Get/Put permissions via permission policy | Apply least privilege principle |
| Add `sts:AssumeRole` permission on Account A's Lambda role | Allow assuming the target role |
| Use temporary credentials from Lambda | Ensure secure access |

---

## 🛠️ Implementation Example

### In Account B: Create IAM Role (ThumbXRole)

**Trust Policy**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::111111111111:role/AccountA-LambdaRole" },
    "Action": "sts:AssumeRole"
  }]
}
```

**Permission Policy**
```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": [
    "arn:aws:s3:::account-b-thumb-bucket",
    "arn:aws:s3:::account-b-thumb-bucket/*"
  ]
}
```

---

### In Account A: Add AssumeRole permission to Lambda role

```json
{
  "Effect": "Allow",
  "Action": "sts:AssumeRole",
  "Resource": "arn:aws:iam::222222222222:role/ThumbXRole"
}
```

---

### Lambda Code: AssumeRole Execution

```python
import boto3
import os

role_arn = os.environ["TARGET_ROLE_ARN"]
sts = boto3.client("sts")
creds = sts.assume_role(
    RoleArn=role_arn,
    RoleSessionName="LambdaCrossAccountSession"
)['Credentials']

s3 = boto3.client(
    's3',
    aws_access_key_id=creds['AccessKeyId'],
    aws_secret_access_key=creds['SecretAccessKey'],
    aws_session_token=creds['SessionToken']
)

# Example: use s3.get_object or s3.put_object with the credentials
```

---

## 🔐 Security Checklist

- Ensure IAM policies follow least privilege principle
- Enable CloudTrail for audit logging
- Configure bucket policies to deny unintended access
- Use Access Analyzer to identify external exposure

---

With this configuration, you can securely build a cross-account Lambda → S3 processing pipeline 💡
