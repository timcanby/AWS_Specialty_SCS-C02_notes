<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [GuardDuty "Impact:IAMUser/AnomalousBehavior" Incident Investigation Design Document](#guardduty-impactiamuseranomalousbehavior-incident-investigation-design-document)
  - [📘 Scenario](#-scenario)
  - [🧠 Key Points](#-key-points)
  - [✅ Recommended Approach](#-recommended-approach)
    - [B. Review the GuardDuty finding using ReadOnly credentials and analyze API behavior in context using Amazon Detective](#b-review-the-guardduty-finding-using-readonly-credentials-and-analyze-api-behavior-in-context-using-amazon-detective)
  - [🚫 Issues with Other Options](#-issues-with-other-options)
  - [🧭 Amazon Detective vs. AWS CloudTrail Insights & CloudTrail Lake](#-amazon-detective-vs-aws-cloudtrail-insights--cloudtrail-lake)
    - [🕵️ Amazon Detective: Role and Use Cases](#-amazon-detective-role-and-use-cases)
      - [🔍 Key Features:](#-key-features)
      - [💡 Best Used For:](#-best-used-for)
    - [📘 AWS CloudTrail Insights & CloudTrail Lake: Role and Use Cases](#-aws-cloudtrail-insights--cloudtrail-lake-role-and-use-cases)
      - [🔍 Key Features:](#-key-features-1)
      - [💡 Best Used For:](#-best-used-for-1)
  - [✅ Summary: Which Tool to Use When?](#-summary-which-tool-to-use-when)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


#  GuardDuty "Impact:IAMUser/AnomalousBehavior" Incident Investigation Design Document

---

## 📘 Scenario

A company hosts a production application in an AWS account.  
Amazon GuardDuty reports the following finding:

> **Impact:IAMUser/AnomalousBehavior** (Anomalous behavior by an IAM user)

The security engineer must **quickly investigate the incident without affecting the production system**, and identify any indications of compromise or abuse.

---

## 🧠 Key Points

| Item | Description |
|------|-------------|
| 🔍 Finding Type | Suspicious API behavior or unusual activity from an IAM user |
| ✅ Avoid Production Impact | Investigation should not interrupt live services |
| 🔐 Least Privilege | Perform analysis using a ReadOnly role wherever possible |
| 📊 Contextual Analysis | It is important to understand not only "what happened" but also "why it's abnormal" |
| ⏱️ Speed | The investigation method must allow for immediate action |

---

## ✅ Recommended Approach

### B. Review the GuardDuty finding using ReadOnly credentials and analyze API behavior in context using Amazon Detective

| Reason | Explanation |
|--------|-------------|
| ✅ No impact on production systems | ReadOnly IAM role allows secure inspection |
| ✅ Use of Amazon Detective | Integrates directly with GuardDuty findings and provides contextual analytics |
| ✅ Visualize user behavior | See what the IAM user did, when, where, and in correlation with other events |

---

## 🚫 Issues with Other Options

| Option | Issue |
|--------|-------|
| A. Add DenyAll policy prematurely | Could block legitimate actions before confirming compromise |
| C. Immediate shutdown as admin | Risky to take disruptive action without investigation |
| D. Use CloudTrail Insights and Lake | Powerful but requires prior setup and offers less immediacy than Detective |

---

## 🧭 Amazon Detective vs. AWS CloudTrail Insights & CloudTrail Lake

Both services are powerful tools for **analyzing anomalous activity and tracing user behavior** in AWS environments.

---

### 🕵️ Amazon Detective: Role and Use Cases

**Amazon Detective** automatically links data from GuardDuty, CloudTrail, and VPC Flow Logs to build **contextual graphs and timelines** around security events.

#### 🔍 Key Features:

- Activity timeline across services
- Correlation between users, IPs, and instances
- Visual graphs for investigation workflow

#### 💡 Best Used For:

- Investigating GuardDuty findings in real time
- Understanding "what else did this IAM user do?"
- Analyzing behavior across accounts or VPCs

---

### 📘 AWS CloudTrail Insights & CloudTrail Lake: Role and Use Cases

**CloudTrail Insights** detects **anomalous API activity** patterns, such as spikes or deviations from normal behavior.  
**CloudTrail Lake** enables long-term storage and **SQL-like querying** of CloudTrail data.

#### 🔍 Key Features:

- Automated detection of API call anomalies (Insights)
- Deep, customizable query access to API logs (Lake)

#### 💡 Best Used For:

- Long-term trend and pattern analysis of API usage
- Detecting subtle or slow-moving threats
- Investigating historical user activity across services

---

## ✅ Summary: Which Tool to Use When?

| Investigation Goal | Best Tool |
|---------------------|-----------|
| Immediate GuardDuty finding analysis | ✅ Amazon Detective |
| Detect deviations in API behavior | ✅ CloudTrail Insights |
| Query deep logs with SQL-like expressions | ✅ CloudTrail Lake |
| Multi-account or region-wide behavioral context | ✅ Amazon Detective |

---

