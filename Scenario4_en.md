<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [AWS CloudFormation StackSets × Security Notification Design Document](#aws-cloudformation-stacksets-%C3%97-security-notification-design-document)
  - [📘 Scenario](#-scenario)
  - [🧠 Key Points](#-key-points)
  - [🛠️ Implementation Example](#-implementation-example)
    - [CloudFormation Guard Rule Samples (for EC2 and S3 validation)](#cloudformation-guard-rule-samples-for-ec2-and-s3-validation)
    - [Example CI/CD Execution (using Docker)](#example-cicd-execution-using-docker)
    - [Example SNS Notification for Validation Failure (CodePipeline)](#example-sns-notification-for-validation-failure-codepipeline)
  - [🔐 Security Checks](#-security-checks)
  - [🚀 Recommended Solution: CloudFormation Guard](#-recommended-solution-cloudformation-guard)
    - [What is CloudFormation Guard (cfn-guard)?](#what-is-cloudformation-guard-cfn-guard)
    - [Common Use Cases](#common-use-cases)
    - [Policy Example](#policy-example)
  - [📦 What is a CloudFormation Stack?](#-what-is-a-cloudformation-stack)
    - [When to Use It](#when-to-use-it)
  - [🛡️ Preventive Measures (Shift-left Security)](#-preventive-measures-shift-left-security)
  - [✅ IaC Best Practices](#-iac-best-practices)
  - [🔔 Conclusion](#-conclusion)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


# AWS CloudFormation StackSets × Security Notification Design Document

---

## 📘 Scenario

A company uses **AWS Organizations** and plans to deploy architectural patterns—including EC2, ELB, RDS, EKS/ECS—across multiple AWS accounts using **CloudFormation StackSets**.  
Currently, developers achieve high delivery speed by creating their own CloudFormation stacks.  
All deployments are orchestrated by a centralized CI/CD pipeline in a shared services account.  
The security team has defined internal security policies for each resource and must be **immediately notified if any resource violates these policies**.  
However, it is essential that the **developers’ delivery speed is not impacted**.

---

## 🧠 Key Points

| Item | Description |
|------|-------------|
| 📦 **StackSets + Organizations** | Enables uniform deployments across multiple accounts. |
| ⚡ **Developer Speed** | The current fast-paced self-service deployment flow must be preserved. |
| 🔐 **Security Compliance** | All deployed resources must adhere to internal security policies. |
| 📨 **Violation Notifications** | Security must be notified in real time upon policy violations. |
| 🎯 **Goal** | Implement security validation and notification with minimal operational overhead. |

---

## 🛠️ Implementation Example

### CloudFormation Guard Rule Samples (for EC2 and S3 validation)

```hcl
rule ec2_instance_type when resourceType == "AWS::EC2::Instance" {
  properties.instanceType == "t3.micro"
}

rule s3_bucket_encryption when resourceType == "AWS::S3::Bucket" {
  properties.BucketEncryption != null
}
```

### Example CI/CD Execution (using Docker)

```bash
docker run --rm \
  -v $(pwd)/template:/template \
  -v $(pwd)/rules:/rules \
  ghcr.io/aws-cloudformation/cloudformation-guard \
  cfn-guard validate -d /template/template.yaml -r /rules
```

### Example SNS Notification for Validation Failure (CodePipeline)

```json
{
  "Action": "Publish",
  "Resource": "arn:aws:sns:ap-northeast-1:123456789012:SecurityNotifications",
  "Condition": {
    "StringEquals": {
      "cfn-guard:Result": "FAILED"
    }
  }
}
```

---

## 🔐 Security Checks

| Item | Description |
|------|-------------|
| ✅ Automated Template Validation | Policy compliance checked in CI/CD using CloudFormation Guard. |
| ✅ Syntax & Dependency Check | Early detection using `cfn-lint`. |
| ✅ Instant Violation Alerts | SNS alerts sent to the security team. |
| ✅ Deployment Blocking | Non-compliant templates are rejected from production deployment. |
| ✅ Repeatability & Consistency | Docker ensures same result across all environments. |

---

## 🚀 Recommended Solution: CloudFormation Guard

### What is CloudFormation Guard (cfn-guard)?

**CloudFormation Guard** is a **policy-as-code tool** that ensures CloudFormation templates comply with enterprise security rules.

| Feature | Description |
|---------|-------------|
| 🔍 Policy DSL | Declarative syntax like `properties.instanceType == "t3.micro"` |
| 🔁 Reusability | Policies can be shared across many templates |
| ⚙️ CI/CD Ready | Easily integrated with CLI or Docker |
| 🔐 Shift-left Security | Issues are detected *before* deployment |

---

### Common Use Cases

| Scenario | Example Policy |
|----------|----------------|
| ✅ Enforce Encryption | Ensure S3, RDS, EBS encryption is enabled |
| ✅ Instance Type Control | Restrict EC2 types (e.g. only allow `t3.*`) |
| ✅ Block Public Access | Deny `0.0.0.0/0` ingress in security groups |
| ✅ IAM Policy Enforcement | Block `Action: "*"` or missing resource scoping |

---

### Policy Example

```hcl
rule ec2_instance_type when resourceType == "AWS::EC2::Instance" {
  properties.instanceType == "t3.micro"
}

rule s3_encryption when resourceType == "AWS::S3::Bucket" {
  properties.BucketEncryption != null
}
```

---

## 📦 What is a CloudFormation Stack?

A **CloudFormation Stack** is a logical unit of deployed AWS resources created from a CloudFormation template.

### When to Use It

| Use Case | Description |
|----------|-------------|
| Multi-resource Setup | Manage and deploy EC2 + RDS + ELB together |
| Reproducible Infra | Templates stored in Git can be redeployed anywhere |
| Multi-account Governance | StackSets enforce uniform structure across the org |

---

## 🛡️ Preventive Measures (Shift-left Security)

| Measure | Example |
|---------|---------|
| 🧾 Define Policies | Enforce encryption, tagging, access scope via Guard |
| 🔄 Linting | Catch syntax and reference errors with `cfn-lint` |
| 🛂 IAM Restrictions | Only allow approved templates in production |
| 🏷️ Tag Enforcement | Require `Owner`, `Environment`, `CostCenter` tags |

---

## ✅ IaC Best Practices

| Practice | Description |
|----------|-------------|
| Parameterize Everything | Avoid hardcoded subnet IDs, AMI IDs |
| Use Nested Stacks | Modularize large templates for clarity |
| Use Change Sets | Confirm changes before applying |
| Git + PR Workflow | Enforce versioning and code reviews |
| Encrypt by Default | Enforce with Guard |
| Avoid Wildcards | Block `"Action": "*"` in IAM policies |

---

## 🔔 Conclusion

By integrating CloudFormation Guard into CI/CD and leveraging SNS for alerting:

- ✅ Templates are checked automatically
- ✅ Developer velocity is preserved
- ✅ Violations are visible immediately
- ✅ Security enforcement is scalable

**This makes CloudFormation Guard the most balanced and efficient solution.**
