#  AWS Organizations × IAM Identity Center (SSO) - Region and Service Restriction Design Guide

---

## 📘 Scenario

A company is designing a multi-account structure using **AWS Organizations** and **IAM Identity Center (formerly AWS SSO)**, assigning one AWS account per development team.  
Requirements:
- Each team should be **restricted to specific AWS regions**  
- Only **specific AWS services should be accessible** in each account  
- **Operational overhead must be minimized**

---

## ✅ Recommended Solution

### **C. Enforce Region and Service Restrictions Using Service Control Policies (SCPs)**

| Reason | Description |
|--------|-------------|
| ✅ Centralized OU-based Control | SCPs can be managed per organizational unit (OU), allowing unified policies across accounts |
| ✅ Higher Priority Than IAM Policies | SCPs override IAM role permissions and explicitly deny actions |
| ✅ Supports Region & Service Control | Use `Condition` or `NotAction` to define allowed regions/services |
| ✅ Scalable | Add accounts to an OU and restrictions are automatically applied |

---

## 🚫 Drawbacks of Other Options

| Option | Drawback |
|--------|----------|
| A. Use Identity Center-only Policies | Requires manual setup per account/role; high management burden |
| B. Disable STS by Region | Ineffective for most services; coarse control |
| D. Custom IAM Policies per Account | Does not scale; complex governance across accounts |

---

## 🧠 Supplemental: Understanding SCPs

### 🔐 What is a Service Control Policy (SCP)?

SCPs are the **top-level access control layer in AWS Organizations**.  
They define the **maximum allowed permissions** per account, overriding IAM permissions.

| Feature | Description |
|---------|-------------|
| Enforceability | IAM permissions allowed = blocked by SCP = not executable |
| Applied by OU | SCPs applied to an OU affect all accounts inside |
| Central Governance | No need to set IAM per account—apply rules centrally |
| Security | Prevent excessive permission grants due to misconfiguration |
| Filterable | `Condition` clauses allow filtering by region or service |

---

## 🌍 Sample SCP for Region Restriction

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOnlySpecificRegions",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": ["ap-northeast-1", "us-west-2"]
        }
      }
    }
  ]
}
```

Blocks operations outside Tokyo and Oregon regions.

---

## 🧰 SCP vs. IAM Policy Differences

| Item | SCP | IAM (Identity Center) |
|------|-----|------------------------|
| Scope | OU-wide (entire organization) | Account × User/Role |
| Scalability | High (instant propagation via OU) | Manual per account |
| Enforcement Type | Allow/Deny (max boundary) | Allow-based execution |
| Applicability | All AWS services and APIs | Only IAM-allowed actions |
| Use Case | Governance across accounts | Per-user permission control |

---

## ✅ Recommended Pattern

- Use SCPs for **common rules** like service/region restrictions  
- Use Identity Center permission sets for **user-specific access scopes**  
- **2-tier model (SCP + Identity Center)** for secure and scalable management

---

## 📘 Setup Example

Assume you're a **new IT admin** at a startup, and today is the first day you're working with your company's brand-new AWS account.  
This section walks through everything you should set up to make your AWS use **secure and production-ready from day one**.

---

### ✅ Step 1: Secure the Root Account

| Task | Description |
|------|-------------|
| Enable MFA on root | Use a virtual MFA app like Google Authenticator |
| Avoid using root | Only for emergencies. Use IAM users or SSO instead |
| Fill in contact/billing info | Input your company info in "Account Settings" |

---

### ✅ Step 2: Create AWS Organizations and Accounts

```bash
1. Open AWS Organizations
2. Create a new Organization
3. Create accounts per purpose: Dev, Test, Prod, etc.
```

Define OUs such as `Dev`, `Test`, `Prod` and group accounts accordingly.

---

### ✅ Step 3: Set Up IAM Identity Center (SSO)

| Task | Example |
|------|---------|
| Enable Identity Center | Go to the console and enable from the management account |
| Create Groups & Users | E.g., `Developers`, `Admins` |
| Assign to Accounts | Assign `ReadOnly` to Developers, `AdminAccess` to Admins |

---

### ✅ Step 4: Enforce Service/Region Restrictions with SCP

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Action": "*",
    "Resource": "*",
    "Condition": {
      "StringNotEquals": {
        "aws:RequestedRegion": ["ap-northeast-1"]
      }
    }
  }]
}
```

This policy denies actions in all regions except Tokyo.

---

### ✅ Step 5: Enable Logging with CloudTrail & GuardDuty

- CloudTrail: Enable in all accounts, send logs to S3 (e.g., LogArchive)
- GuardDuty: Enable in all accounts, aggregate to a central security account

---

### ✅ Step 6: Monitor Cost and Budget

| Tool | Description |
|------|-------------|
| AWS Budgets | Set monthly limits, receive alerts via SNS |
| Cost Explorer | View usage by service/account |
| Billing Alerts | SNS + CloudWatch thresholds for billing spikes |

---

### ✅ Step 7: Best Practices

- Use **least privilege** IAM policy design
- Avoid access keys; prefer **SSO or MFA**
- Don’t use default VPCs for prod; design networks intentionally
- Enforce tags: `Name`, `Environment`, `Owner`

---

### 📌 Summary: Initial Setup Template

| Item | Recommended |
|------|-------------|
| Account Structure | Organizations + OU + Identity Center |
| Access Management | IAM Identity Center + Permission Sets |
| Logging | CloudTrail + GuardDuty |
| Governance | SCP for region/service control |
| Cost Control | Budgets + Cost Explorer |

---

This setup provides a **strong foundation** for security, scalability, and clarity in your company’s AWS usage.

