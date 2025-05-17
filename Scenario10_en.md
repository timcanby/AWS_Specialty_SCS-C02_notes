<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Cross-Account S3 Access Design Document (Account A → Account B)](#cross-account-s3-access-design-document-account-a-%E2%86%92-account-b)
  - [📘 Scenario](#-scenario)
  - [🧠 Key Points](#-key-points)
  - [✅ Solution: Bucket Policy in Account B](#-solution-bucket-policy-in-account-b)
    - [Example Bucket Policy in Account B](#example-bucket-policy-in-account-b)
  - [📘 Required Conditions for Cross-Account Access](#-required-conditions-for-cross-account-access)
  - [🚫 Common Misconfigurations](#-common-misconfigurations)
  - [🛡️ Best Practices for Cross-Account S3 Access](#-best-practices-for-cross-account-s3-access)
  - [✅ Conclusion](#-conclusion)
  - [🧾 Example IAM Policy in Account A (Accessing User)](#-example-iam-policy-in-account-a-accessing-user)
  - [📝 Troubleshooting Checklist](#-troubleshooting-checklist)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


#  Cross-Account S3 Access Design Document (Account A → Account B)

---

## 📘 Scenario

- Company A (Account A) has acquired Company B (Account B).
- Company B stores files in an S3 bucket.
- Company A's IAM user needs **full access** to the S3 bucket in Account B.
- Although the IAM policy is configured on the Account A side, access is still denied.

---

## 🧠 Key Points

| Item | Description |
|------|-------------|
| ✅ IAM policy alone is not enough | IAM defines who is allowed to act, but the S3 bucket must also accept the access |
| ✅ Use resource-based policy on the S3 bucket | A bucket policy in Account B must explicitly allow access to the external IAM user |
| ✅ Both Principal and Resource must be defined | You need to declare both sides using proper ARNs |
| ❌ ACL is deprecated | ACLs are considered legacy and not recommended for fine-grained control |
| ❌ User policy in Account B is ineffective | Cannot affect IAM users from Account A |

---

## ✅ Solution: Bucket Policy in Account B

### Example Bucket Policy in Account B

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111122223333:user/TargetUser"
      },
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::example-bucket",
        "arn:aws:s3:::example-bucket/*"
      ]
    }
  ]
}
```

- `Principal`: The IAM user in Account A (the requester)
- `Resource`: The S3 bucket and all objects
- `Action`: Adjust as needed (e.g., `s3:GetObject`, `s3:PutObject`, or `s3:*`)

---

## 📘 Required Conditions for Cross-Account Access

| Condition | Description |
|-----------|-------------|
| ① IAM policy on the accessing user | Must allow access to the specific bucket and actions |
| ② Bucket policy in the target account | Must explicitly permit access to the IAM principal |

---

## 🚫 Common Misconfigurations

| Method | Issue |
|--------|-------|
| Bucket ACL | Hard to manage, limited granularity, deprecated |
| Object ACL | Must be set individually per object; not scalable |
| User policy in Account B | Cannot control access from Account A's users |

---

## 🛡️ Best Practices for Cross-Account S3 Access

- Use **resource-based bucket policies**
- Define **explicit ARNs** (user or role from the other account)
- Follow **principle of least privilege**
- Enable logging (e.g., S3 access logs, AWS CloudTrail)

---

## ✅ Conclusion

- Both **IAM policy** and **bucket policy** must be configured to enable cross-account access.
- The bucket owner (Account B) must **explicitly allow the external principal**.
- Therefore, the correct solution is **C: Add a bucket policy in Account B**.

---

## 🧾 Example IAM Policy in Account A (Accessing User)

To give the IAM user in Account A the right permissions, attach the following IAM policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::example-bucket",
        "arn:aws:s3:::example-bucket/*"
      ]
    }
  ]
}
```

- `Action`: Limit to `s3:GetObject`, `s3:PutObject` as needed
- `Resource`: Use the full ARN of the target bucket in Account B
- Note: Even with this policy, **access will be denied unless Account B allows it via a bucket policy**

---

## 📝 Troubleshooting Checklist

1. Double-check the `Principal` ARN in the bucket policy
2. Ensure both IAM and bucket policies include all required `Action` permissions
3. Confirm bucket owner settings do not block ACLs (e.g., "Object Ownership: Bucket owner enforced")
4. Use IAM Policy Simulator or S3 Access Analyzer to validate permissions

---
