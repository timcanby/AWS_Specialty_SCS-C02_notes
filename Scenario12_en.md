<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [EC2 Auto Scaling × Long-Term Log Retention Design (New Region)](#ec2-auto-scaling-%C3%97-long-term-log-retention-design-new-region)
  - [📘 Scenario](#-scenario)
  - [🧠 Design Considerations](#-design-considerations)
  - [🛠️ Components and Recommended Implementation](#-components-and-recommended-implementation)
    - [1. Install and configure the CloudWatch Agent on all EC2 instances](#1-install-and-configure-the-cloudwatch-agent-on-all-ec2-instances)
    - [2. Set log retention policy for CloudWatch Logs to 7 years](#2-set-log-retention-policy-for-cloudwatch-logs-to-7-years)
    - [3. Attach an IAM Role with log write permissions to EC2 instances](#3-attach-an-iam-role-with-log-write-permissions-to-ec2-instances)
  - [🔐 Security and Operational Considerations](#-security-and-operational-considerations)
  - [📊 Monitoring Log Delivery in Auto Scaling Environments](#-monitoring-log-delivery-in-auto-scaling-environments)
    - [✅ Monitoring Methods](#-monitoring-methods)
      - [1. Use CloudWatch Logs Insights to query log delivery](#1-use-cloudwatch-logs-insights-to-query-log-delivery)
      - [2. Set up Metric Filters in CloudWatch Logs](#2-set-up-metric-filters-in-cloudwatch-logs)
      - [3. Use Auto Scaling Lifecycle Hooks for log validation](#3-use-auto-scaling-lifecycle-hooks-for-log-validation)
      - [4. Monitor using CloudWatch Alarms](#4-monitor-using-cloudwatch-alarms)
  - [🔁 Operational Tips](#-operational-tips)
  - [✅ Conclusion](#-conclusion)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


#  EC2 Auto Scaling × Long-Term Log Retention Design (New Region)

---

## 📘 Scenario

A global company has established a new AWS account and business presence in South Korea (ap-northeast-2).  
The company is operating workloads using three Auto Scaling groups with EC2 instances.  
All system and application logs generated in this region must be retained for **7 years**.  
The log collection system must ensure **no data is lost**, even during instance scaling (launch/terminate).

---

## 🧠 Design Considerations

| Requirement | Description |
|-------------|-------------|
| ✅ Durable log storage | Logs must be sent off-instance immediately to survive instance termination |
| ✅ Automatic retention enforcement | Logs older than 7 years should be deleted automatically |
| ✅ Scalable, consistent structure | The setup should apply uniformly across all Auto Scaling groups |

---

## 🛠️ Components and Recommended Implementation

### 1. Install and configure the CloudWatch Agent on all EC2 instances

- Configure the CloudWatch Agent to send system and application logs to CloudWatch Logs
- Include configuration in Launch Templates or via user-data scripts for Auto Scaling groups

### 2. Set log retention policy for CloudWatch Logs to 7 years

- Set retention period of **2555 days** on log groups
- Prevents log over-accumulation and ensures compliance with data lifecycle requirements

### 3. Attach an IAM Role with log write permissions to EC2 instances

- Define an IAM role and attach it to Auto Scaling Launch Templates
- Role must allow log delivery (`logs:PutLogEvents`, etc.) to CloudWatch Logs

---

## 🔐 Security and Operational Considerations

- Prevent log tampering and support auditing via CloudTrail and CloudWatch Logs Insights
- Manage agent configuration versions centrally (e.g., in S3 or Systems Manager Parameter Store)
- Monitor Auto Scaling events and ensure log delivery through CloudWatch Metrics or alarms

---

## 📊 Monitoring Log Delivery in Auto Scaling Environments

To ensure that logs are **consistently delivered to CloudWatch Logs**, even during frequent scale-out/scale-in activity, the following practices are recommended:

### ✅ Monitoring Methods

#### 1. Use CloudWatch Logs Insights to query log delivery

- Detect if logs are received from all expected instances over a set time range
- Example: detect missing logs in the past 30 minutes

```sql
fields @timestamp, @logStream
| filter @timestamp > ago(30m)
| stats count() by @logStream
```

- Can be automated with EventBridge Scheduler + Lambda for periodic checking and Slack/email alerts

#### 2. Set up Metric Filters in CloudWatch Logs

- Match known patterns such as "agent started", and count entries per log stream
- Create CloudWatch Alarms when no logs are seen for a given time threshold

#### 3. Use Auto Scaling Lifecycle Hooks for log validation

- Add lifecycle hook on instance launch to verify agent startup and first log delivery
- Use SNS + Lambda to handle lifecycle events and record validation results

#### 4. Monitor using CloudWatch Alarms

- Create alarms for “no log delivery for 15+ minutes”
- Integrate with SNS or AWS Incident Manager for alert routing

---

## 🔁 Operational Tips

- Use IaC (CloudFormation/Terraform) to maintain log monitoring consistency
- In high-scale environments, compare desired capacity with number of active log streams
- Periodically run a script to cross-check instance count with log stream count for drift detection

---

## ✅ Conclusion

This setup enables reliable, long-term logging for all EC2 instances under Auto Scaling, fully aligned with 7-year compliance requirements and operational scalability in the ap-northeast-2 region.
