<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Multi-Account Authentication and Authorization Design: A Scalable and Native Approach](#multi-account-authentication-and-authorization-design-a-scalable-and-native-approach)
  - [📘 Scenario](#-scenario)
  - [🧠 Key Points](#-key-points)
  - [✅ Recommended Approach (Design Principles)](#-recommended-approach-design-principles)
    - [1. Use the Default Directory in IAM Identity Center](#1-use-the-default-directory-in-iam-identity-center)
    - [2. Map Groups to Permission Sets and Assign to Accounts](#2-map-groups-to-permission-sets-and-assign-to-accounts)
    - [3. Enable Access via IAM Identity Center User Portal](#3-enable-access-via-iam-identity-center-user-portal)
  - [🚫 Approaches to Avoid](#-approaches-to-avoid)
    - [1. Using External or On-Premises Directories](#1-using-external-or-on-premises-directories)
    - [2. Direct Mapping to IAM Users](#2-direct-mapping-to-iam-users)
  - [🛠️ Implementation Example](#-implementation-example)
    - [Initial Setup with IAM Identity Center](#initial-setup-with-iam-identity-center)
  - [📌 Conclusion](#-conclusion)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


#  Multi-Account Authentication and Authorization Design: A Scalable and Native Approach

---

## 📘 Scenario

A company manages multiple AWS accounts using AWS Organizations and has enabled IAM Identity Center (formerly AWS SSO).  
A security engineer is tasked with designing a centralized authentication and authorization solution for all employees.

- The company prefers **not to introduce additional user-managed infrastructure** (e.g., Active Directory)
- Access should be granted based on **employees' job functions**
- The system must be **scalable and centrally managed**

---

## 🧠 Key Points

| Item | Description |
|------|-------------|
| ✅ Fully Managed | IAM Identity Center is a native AWS managed service requiring no additional setup |
| ✅ Centralized User Management | Users and groups can be managed directly within IAM Identity Center |
| ✅ Job-Based Authorization | Use Permission Sets linked to groups based on roles such as developer, auditor, etc. |
| ✅ User Portal | Users access their assigned AWS accounts and roles via a dedicated Identity Center portal |
| ✅ Integration with AWS Organizations | Permissions can be centrally managed across newly added accounts |

---

## ✅ Recommended Approach (Design Principles)

### 1. Use the Default Directory in IAM Identity Center

- Create and manage users and groups within IAM Identity Center
- Avoid dependency on external directories like AD Connector or AWS Managed Microsoft AD

### 2. Map Groups to Permission Sets and Assign to Accounts

- Define permission sets according to job functions (e.g., Admin, Dev, ReadOnly)
- Map each group to the corresponding AWS accounts using IAM Identity Center

### 3. Enable Access via IAM Identity Center User Portal

- Employees log in through the IAM Identity Center portal
- Easily switch roles and accounts with SSO and centralized session control

---

## 🚫 Approaches to Avoid

### 1. Using External or On-Premises Directories

- AD Connector and AWS Managed Microsoft AD require setup, maintenance, and networking
- Contradicts the requirement to avoid additional infrastructure

### 2. Direct Mapping to IAM Users

- IAM Identity Center does not integrate with IAM users directly
- Role-based access through Permission Sets is the modern and recommended pattern

---

## 🛠️ Implementation Example

### Initial Setup with IAM Identity Center

1. Enable AWS Organizations with all features
2. Activate IAM Identity Center and select the default directory
3. Create users and groups in IAM Identity Center
4. Define permission sets for each group based on job function
5. Assign permission sets to the appropriate AWS accounts

---

## 📌 Conclusion

- ✅ By leveraging **AWS-native services like IAM Identity Center**, you can build a **centralized, scalable, and secure identity system** without the overhead of managing external infrastructure.
- ✅ This approach supports low maintenance, job-function-based access control, and seamless multi-account governance with minimal cost.
