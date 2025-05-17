<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [AWS Security Hub Critical Finding Email Notification Design Document](#aws-security-hub-critical-finding-email-notification-design-document)
  - [📘 Scenario](#-scenario)
  - [🧠 Key Points](#-key-points)
  - [✅ Best Architecture (Correct Answer)](#-best-architecture-correct-answer)
    - [C. Use an EventBridge rule to filter for critical findings and send to SNS for email delivery](#c-use-an-eventbridge-rule-to-filter-for-critical-findings-and-send-to-sns-for-email-delivery)
    - [🎯 Example EventBridge Rule Pattern:](#-example-eventbridge-rule-pattern)
  - [🚫 Issues With Other Options](#-issues-with-other-options)
  - [📌 Conclusion](#-conclusion)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


#  AWS Security Hub Critical Finding Email Notification Design Document

---

## 📘 Scenario

A company wants to receive **email notifications when AWS Security Hub detects a "CRITICAL" finding**.  
The company currently has no infrastructure to support this function.

---

## 🧠 Key Points

| Item | Description |
|------|-------------|
| ✅ Security Hub integrates with EventBridge | Security Hub can send findings to EventBridge in near real time |
| ✅ SNS is ideal for email notification | Amazon SNS can directly send emails to subscribed endpoints |
| ✅ Simplicity matters | The goal is just to send an email, so avoid complex architectures |

---

## ✅ Best Architecture (Correct Answer)

### C. Use an EventBridge rule to filter for critical findings and send to SNS for email delivery

| Step | Description |
|------|-------------|
| ① | Create an EventBridge rule to filter findings with `SeverityLabel = "CRITICAL"` |
| ② | Set Amazon SNS topic as the rule's target |
| ③ | Subscribe an email address to the SNS topic |
| ④ | Confirm the subscription via email to begin receiving alerts |

---

### 🎯 Example EventBridge Rule Pattern:

```json
{
  "source": ["aws.securityhub"],
  "detail-type": ["Security Hub Findings - Imported"],
  "detail": {
    "findings": {
      "Severity": {
        "Label": ["CRITICAL"]
      }
    }
  }
}
```

---

## 🚫 Issues With Other Options

| Option | Issue |
|--------|-------|
| A. Lambda → SNS | Overcomplicated — EventBridge can directly call SNS |
| B. Kinesis Data Firehose | Intended for log delivery, not email notifications |
| D. Amazon SES | Requires domain validation and setup, overkill for this case; SNS is simpler and faster |

---

## 📌 Conclusion

- ✅ The most practical and simplest architecture is:  
  **EventBridge rule (filtering CRITICAL) → SNS topic → Email**
- ✅ No code or infrastructure setup required beyond native AWS service configuration
