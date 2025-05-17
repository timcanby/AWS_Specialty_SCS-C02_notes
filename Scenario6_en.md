<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Amazon DynamoDB × Backup and Retention Policy Compliance Design Document](#amazon-dynamodb-%C3%97-backup-and-retention-policy-compliance-design-document)
  - [📘 Scenario](#-scenario)
  - [🧠 Key Points](#-key-points)
  - [✅ Recommended Approach (Backup and Retention Strategy)](#-recommended-approach-backup-and-retention-strategy)
    - [1. Create a Backup Plan Using AWS Backup](#1-create-a-backup-plan-using-aws-backup)
    - [2. Automate Scheduled Backups Using cron Expressions](#2-automate-scheduled-backups-using-cron-expressions)
  - [🚫 Methods to Avoid](#-methods-to-avoid)
    - [Manual Use of On-Demand Backups](#manual-use-of-on-demand-backups)
    - [Using AWS DataSync](#using-aws-datasync)
  - [🛠️ Implementation Example](#-implementation-example)
    - [AWS Backup Plan Definition (JSON)](#aws-backup-plan-definition-json)
    - [Assigning the Backup Plan to DynamoDB Tables](#assigning-the-backup-plan-to-dynamodb-tables)
  - [🔁 Restore Procedure](#-restore-procedure)
    - [Steps to Restore from AWS Backup](#steps-to-restore-from-aws-backup)
    - [Restore Considerations](#restore-considerations)
  - [🔐 Security and Audit Considerations](#-security-and-audit-considerations)
  - [📌 Conclusion](#-conclusion)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


#  Amazon DynamoDB × Backup and Retention Policy Compliance Design Document

---

## 📘 Scenario

A company uses **dozens of Amazon DynamoDB tables** to store application data.  
An audit revealed that the **current backup settings do not comply with the internal data protection policy**.

The company's backup and retention policy is as follows:

- **Backups must be created at midnight on the 15th and 25th of each month**
- **Each backup must be retained for 3 months**

---

## 🧠 Key Points

| Item | Description |
|------|-------------|
| 📅 Backup Schedule | Run twice a month (15th and 25th) at 00:00 |
| ⏳ Retention Period | Retain each backup for 90 days |
| 🗃️ Target Resources | Set of DynamoDB tables |
| ⚠️ Compliance Violation | Current backups are manual or irregular, with no retention settings |
| 🎯 Goal | Implement automated scheduling and retention rules for audit compliance |

---

## ✅ Recommended Approach (Backup and Retention Strategy)

To meet the backup policy requirements for Amazon DynamoDB, implement the following:

### 1. Create a Backup Plan Using AWS Backup

- AWS Backup natively supports DynamoDB and allows centralized management of multiple tables.
- By configuring a backup rule with a **retention period of 90 days (3 months)**, the solution aligns with internal policy.
- The execution schedule, retention period, and restore operations can all be centrally managed.

### 2. Automate Scheduled Backups Using cron Expressions

- Set the AWS Backup schedule expression to **cron(0 0 15,25 * ? *)** to execute backups precisely at midnight on the 15th and 25th of each month.
- `rate` expressions cannot specify exact calendar days, so **cron is required**.
- Assign all DynamoDB tables to the backup plan to enable fully automated, recurring protection.

---

## 🚫 Methods to Avoid

### Manual Use of On-Demand Backups

- DynamoDB on-demand backup is **manual by default**, lacking automatic scheduling or retention management.
- This approach is unsuitable for scalable backup strategies and increases operational burden.

### Using AWS DataSync

- AWS DataSync is primarily for transferring files between S3 and file servers.  
  **It is not suitable for backing up DynamoDB.**

---

## 🛠️ Implementation Example

### AWS Backup Plan Definition (JSON)

```json
{
  "BackupPlanName": "DynamoDBMonthlyBackup",
  "Rules": [
    {
      "RuleName": "TwiceMonthly",
      "TargetBackupVaultName": "Default",
      "ScheduleExpression": "cron(0 0 15,25 * ? *)",
      "StartWindowMinutes": 60,
      "CompletionWindowMinutes": 180,
      "Lifecycle": {
        "DeleteAfterDays": 90
      }
    }
  ]
}
```

### Assigning the Backup Plan to DynamoDB Tables

- Create a **Backup selection** in the AWS Backup console or via API.
- Specify the resource type as `AWS::DynamoDB::Table` and add all target tables.

---

## 🔁 Restore Procedure

### Steps to Restore from AWS Backup

1. Open the AWS Backup console
2. Select the target backup vault
3. Choose the relevant backup snapshot
4. Click on “Restore”
5. Specify the destination region and table name
6. Start the restore process

### Restore Considerations

| Consideration | Detail |
|---------------|--------|
| Duplicate Table Names | A different name must be used if restoring into the same environment |
| Restore Duration | Varies by table size; may take minutes to hours |
| Cost | Restored data is billed as new storage |

---

## 🔐 Security and Audit Considerations

| Item | Description |
|------|-------------|
| ✅ Centralized Backup Management | Manage schedule and retention with AWS Backup |
| ✅ Compliance Alignment | Clear retention rules meet policy requirements |
| ✅ Audit Logs | Use AWS CloudTrail to track backup operations |
| ✅ Automation | Implement with IaC tools like Terraform or CloudFormation |
| ✅ Cost Optimization | Automatically delete backups after 3 months |

---

## 📌 Conclusion

- ✅ **Using AWS Backup + cron scheduling + retention rules** enables a fully compliant, secure, and maintainable backup strategy for DynamoDB.
