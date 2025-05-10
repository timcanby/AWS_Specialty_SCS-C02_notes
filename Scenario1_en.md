
# AWS Lambda × S3 Secure Design for Serverless Image Processing Systems

## 📘 Scenario

A company is building a system that uses **AWS Lambda functions to generate thumbnail images from large images**.  
The Lambda function accesses an **S3 bucket within the same AWS account**, reads the original image, and saves the generated thumbnail.

This architecture is a **typical event-driven serverless AWS pattern**, where **S3 upload events trigger the Lambda function, and the processed result is written back to S3**.

In such cases, it is crucial to configure **secure and proper permission settings to allow Lambda to access S3**.

---

## 🎯 Test Points

- How to securely configure access permissions from Lambda to S3
- Understanding IAM fundamentals (IAM users, IAM roles, IAM policies)
- How S3 bucket policy works and how it integrates with IAM policy
- AWS security best practices that avoid long-term credentials
- Difference between IAM/bucket policy and network-based control (e.g., Security Group)

---

## 🧠 Related Knowledge

### IAM User vs. IAM Role

- IAM users are for individuals or applications and use long-term access keys.
- IAM roles provide temporary credentials to AWS services (e.g., Lambda, EC2), and are **ideal for service-to-service access**.
- Assigning an IAM role to a Lambda function allows it to securely access other AWS services such as S3.

### IAM Policy

- IAM policies define "which actions are allowed on which resources."
- When attached to an IAM role, the policy enables Lambda to perform actions like `s3:GetObject` or `s3:PutObject`.

### S3 Bucket Policy

- A resource-based policy that can be attached directly to an S3 bucket.
- By specifying the IAM role as a principal, you can manage access from the bucket side independently from the Lambda-side IAM role.
- Used together with IAM policies, it enables fine-grained access control.

### AWS Secrets Manager (reference)

- A service for securely storing secrets such as passwords or API keys.
- **Not appropriate for storing EC2 key pairs to access S3.**

### EC2 Key Pair

- Authentication credentials used to log in to EC2 via SSH.
- **Not used for accessing the S3 API.**

### Security Group

- Network-level (L3/L4) access control (based on IP/port).
- **Does not control access to S3 APIs; IAM and bucket policies are used for that.**

---

## ✅ Summary

This scenario is a representative data processing pattern in serverless architecture.  
It requires **proper IAM design to securely and reliably allow Lambda access to S3**.

### Recommended Approach

- ✅ Assign an IAM role to Lambda and attach a policy granting S3 access
- ✅ Define a bucket policy on S3 to allow access from the IAM role used by Lambda

### Approaches to Avoid

- ❌ Embedding IAM user access keys into Lambda (long-term credential usage)
- ❌ Using Secrets Manager to store EC2 key pairs for S3 access
- ❌ Trying to control S3 access using Security Groups

In Lambda and S3 integration, **the best practice is to design around IAM roles**.

---

## IAM Roles: Trust Policy vs. Permissions Policy

| Type             | Target           | Purpose                                               | Who can assume this role?                  | What can this role do?                   |
|------------------|------------------|--------------------------------------------------------|--------------------------------------------|------------------------------------------|
| **Trust Policy** | The IAM Role     | Defines **who can assume this role**                  | ✅ e.g., Lambda service                     | ❌ Does not include resource permissions   |
| **Permissions Policy** | The IAM Role | Defines **what actions this role can perform**         | ❌ Does not control who can assume it       | ✅ e.g., S3 access, CloudWatch logs        |

### 💡 Easy Analogy

- **Trust Policy** = Who can "wear" the role (like lending a uniform)
- **Permissions Policy** = What the "wearer" of the role is allowed to do (like access badge permissions)

---

### Example: Lambda Execution Role

- Trust policy allows `lambda.amazonaws.com` to assume the role
- Permissions policy grants access to CloudWatch Logs and S3 buckets

Only when both are set up properly can the role access resources securely.

---

# Lambda × S3: Centralized Access Control via Bucket Policy

## ✅ Step 1: Create the Lambda Execution Role (e.g., LambdaThumbRole)

### 🔸 Trust Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "lambda.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### 🔸 Minimal IAM Policy (for logging only)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

This policy grants only the minimum permission to write logs to CloudWatch Logs.  
※ S3 access is controlled by the bucket policy.

---

## ✅ Step 2: Add a Bucket Policy to the S3 bucket

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowLambdaThumbRoleAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/LambdaThumbRole"
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::my-image-bucket",
        "arn:aws:s3:::my-image-bucket/*"
      ]
    }
  ]
}
```

---

## ✅ Step 3 (Optional): Trigger Lambda from S3 Event

In S3 "Event Notification" settings, configure `ObjectCreated` events to trigger your Lambda function.  
➡ This ensures that the Lambda is automatically invoked when a file is uploaded, and generates the thumbnail.

---

# ✅ Summary: Key Points on Lambda Role and S3 Bucket Policy Permissions

## 💡 Background

- The Lambda execution role (LambdaThumbRole) has **only logging permissions**:
  ```json
  {
    "Effect": "Allow",
    "Action": [
      "logs:CreateLogGroup",
      "logs:CreateLogStream",
      "logs:PutLogEvents"
    ],
    "Resource": "*"
  }
  ```
- **No S3 permissions (`GetObject` / `PutObject`) are granted.**

## 💡 Why can Lambda still access S3?

Because a **bucket policy (resource policy)** like the one below is added:

```json
{
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:role/LambdaThumbRole"
  },
  "Action": [
    "s3:GetObject",
    "s3:PutObject"
  ],
  "Effect": "Allow",
  "Resource": "arn:aws:s3:::my-image-bucket/*"
}
```

So, **S3 grants access based on its own resource-side permission**, even though the IAM role lacks explicit S3 permissions.

---

## ✅ Conclusion: For some AWS services (e.g., S3), access can be granted solely by the resource policy

| Condition                                            | Result             |
|------------------------------------------------------|--------------------|
| IAM role has no S3 permission, bucket policy allows  | ✅ Access allowed   |
| IAM role explicitly denies S3 access                 | ❌ Access denied    |
| Bucket policy does not allow the principal           | ❌ Access denied    |

---

## 🎯 Key Takeaways

- Deliberately avoid granting S3 access in Lambda’s IAM role (minimal permissions)
- Use bucket policy to centrally manage access on the S3 side
- AWS evaluates access as `IAM Policy ∩ Resource Policy`, but in some services like S3, **resource policy alone can authorize access**

---

## 🔐 Security Benefits

- Prevents unintended access to S3 buckets
- Centralized access control by resource administrators
- Suitable for environments requiring auditability and visibility

---

## ✅ This is a textbook example of centralized access control on the resource side

When integrating Lambda with S3,  
assign only log permissions to Lambda’s role, and grant S3 access explicitly via bucket policy.  
This results in **a more secure and well-audited permission design**.
