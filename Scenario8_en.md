<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [GuardDuty × Automated Incident Containment Design Document (RDP Brute Force Attack Response)](#guardduty-%C3%97-automated-incident-containment-design-document-rdp-brute-force-attack-response)
  - [📘 Scenario](#-scenario)
  - [🧠 Key Points](#-key-points)
  - [🧩 What is an RDP Brute Force Attack?](#-what-is-an-rdp-brute-force-attack)
    - [🔎 Technical Behavior Examples](#-technical-behavior-examples)
    - [🧪 Examples of GuardDuty Detection Scenarios](#-examples-of-guardduty-detection-scenarios)
    - [⚠️ Why It’s Dangerous](#-why-its-dangerous)
  - [✅ Recommended Design Approach](#-recommended-design-approach)
  - [🚫 Methods to Avoid](#-methods-to-avoid)
  - [🛠️ Implementation Example](#-implementation-example)
    - [EventBridge + Lambda Blocking Workflow](#eventbridge--lambda-blocking-workflow)
  - [🧭 What is AWS Security Hub?](#-what-is-aws-security-hub)
    - [🔍 Key Features:](#-key-features)
    - [💡 Common Use Cases](#-common-use-cases)
  - [🔍 What to Do After Blocking the Attack](#-what-to-do-after-blocking-the-attack)
    - [🔬 1. Forensic Analysis of the Instance](#-1-forensic-analysis-of-the-instance)
    - [🧾 2. Analyze CloudTrail and VPC Flow Logs](#-2-analyze-cloudtrail-and-vpc-flow-logs)
    - [📦 3. AMI/Image Audit and Rebuild](#-3-amiimage-audit-and-rebuild)
    - [📘 4. Monitor Trends and Classify Findings](#-4-monitor-trends-and-classify-findings)
    - [🛡️ 5. Review Security Policies and Monitoring Rules](#-5-review-security-policies-and-monitoring-rules)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


#  GuardDuty × Automated Incident Containment Design Document (RDP Brute Force Attack Response)

---

## 📘 Scenario

A company has enabled **Amazon GuardDuty** and now wants to implement automated, temporary blocking measures for threats it detects.

The initial target use case is:

- **RDP (Remote Desktop Protocol) brute force attacks originating from Amazon EC2 instances**
- The attack source is assumed to be an EC2 instance within the company’s own environment
- Communication from the suspicious instance should be blocked temporarily to allow time for investigation and remediation

---

## 🧠 Key Points

| Item | Description |
|------|-------------|
| ✅ Leverage GuardDuty Findings | GuardDuty detects threats like RDP brute force attacks in near real time |
| ✅ Automatic Containment | Block communication automatically in response to findings |
| ✅ Temporary Blocking Methods | Use security groups, network ACLs, or Network Firewall rules |
| ✅ Room for Manual Investigation | Blocking should be non-destructive and allow recovery after investigation |
| ✅ Use AWS Native Services | Combine AWS managed services like GuardDuty, EventBridge, Lambda, SNS for orchestration |

---

## 🧩 What is an RDP Brute Force Attack?

- A “brute force attack” refers to repeatedly trying all possible username and password combinations to gain unauthorized access.
- RDP (Remote Desktop Protocol) is commonly used to connect remotely to Windows or Linux servers for remote work or system administration.
- An “RDP brute force attack” is a cyberattack that targets TCP port 3389 with repeated login attempts using common or randomized credentials.

---

### 🔎 Technical Behavior Examples

- Dozens of login attempts per second are sent from the attacking EC2 instance to external or internal systems.
- Common usernames like “Administrator”, “Admin”, “root”, “Guest”, “test” are used repeatedly.
- TCP 3389 connection attempts are observed to IPs such as:
  - `203.0.113.10`, `203.0.113.11`, `198.51.100.22`, etc.
- In some cases, a PowerShell script may be downloaded remotely after a successful login.
- GuardDuty analyzes these patterns and generates a finding like:

```json
{
  "type": "BruteForce:RDP/RemoteHost",
  "resource": {
    "instanceDetails": {
      "instanceId": "i-0123456789abcdef0",
      "tags": [...],
      "networkInterfaces": [...]
    }
  },
  "service": {
    "action": {
      "portProbeAction": {
        "portProbeDetails": [
          {
            "localPortDetails": {
              "port": 3389,
              "portName": "RDP"
            },
            "remoteIpDetails": {
              "ipAddressV4": "203.0.113.45",
              "organization": {
                "asn": 64512,
                "isp": "Suspicious ISP"
              }
            }
          }
        ]
      }
    }
  }
}
```

---

### 🧪 Examples of GuardDuty Detection Scenarios

1. **RDP attacks detected from a developer EC2 instance to external hosts**
   - Cause: A base AMI used internally contained malware
   - Response: Isolated the instance and rebuilt the image

2. **Bastion host repeatedly attempted RDP logins to another VPC**
   - Cause: Misconfigured security groups allowed internal exposure
   - Response: Immediate SG correction and log investigation

3. **Marketplace Windows Server showed outbound RDP attempts**
   - Cause: Exposed test environment misused as an attack tool
   - Response: GuardDuty triggered auto-isolation and incident review

---

### ⚠️ Why It’s Dangerous

- If an EC2 instance is the source of such attacks, it may be compromised (e.g., malware infection or credentials leak).
- Immediate isolation is essential to prevent lateral movement or further harm.
- Ignoring such activity can lead to IP blacklisting, trust damage, or even AWS account suspension.

---

## ✅ Recommended Design Approach

- Use **Amazon EventBridge** to receive GuardDuty findings
- Replace the instance’s security group with one that blocks all traffic
- Use Lambda for automation and SNS for alerting
- Allows safe temporary containment and easy manual investigation

---

## 🚫 Methods to Avoid

| Method | Reason |
|--------|--------|
| A. Kinesis + ACL | Too complex and inflexible; ACLs are IP-based and less suited for EC2 instance-level control |
| B. AWS WAF | WAF operates at Layer 7 (HTTP/HTTPS), not suitable for Layer 4 RDP traffic |
| C. Network Firewall | Powerful but overly complex and costly for this simple, urgent use case |

---

## 🛠️ Implementation Example

### EventBridge + Lambda Blocking Workflow

1. GuardDuty generates an `RDPBruteForce` finding  
2. EventBridge rule triggers on the finding
3. Lambda function:
   - Retrieves and saves the current security group
   - Replaces it with a restrictive "deny-all" security group
4. SNS sends a notification to the security team

---

## 🧭 What is AWS Security Hub?

**AWS Security Hub** aggregates, analyzes, and visualizes security findings from AWS services (GuardDuty, Inspector, Macie, Firewall Manager) and third-party tools.

### 🔍 Key Features:

- **Unified Dashboard**: View security status across accounts and regions
- **Standards-Based Checks**: Includes benchmarks like CIS AWS Foundations
- **EventBridge Integration**: Automatically trigger remediation actions
- **ASFF Format**: Unified JSON format to normalize findings from multiple sources

### 💡 Common Use Cases

- Centralize GuardDuty findings to prioritize high-severity threats
- Automate blocking/remediation via EventBridge + Lambda
- Monitor security posture across multi-account AWS organizations

---

## 🔍 What to Do After Blocking the Attack

Even after blocking the suspicious instance, **analyzing the root cause and impact is critical** for long-term protection and prevention.

### 🔬 1. Forensic Analysis of the Instance

- **Memory dump capture** using EC2 tools
- **Create disk snapshots** for malware scanning in isolated environments
- **Check login and process history** (Windows Event Viewer, Linux `auth.log`, bash history)

### 🧾 2. Analyze CloudTrail and VPC Flow Logs

- Use CloudTrail to see **who did what and when**
- Review VPC Flow Logs for **unusual outbound traffic** from the instance

### 📦 3. AMI/Image Audit and Rebuild

- Re-examine the base AMI or custom image used by the instance
- If backdoors or malware are found, update the AMI and consider signing

### 📘 4. Monitor Trends and Classify Findings

- Check if similar attacks are happening from other instances
- Use Security Hub to compare historical attack patterns and classify severity

### 🛡️ 5. Review Security Policies and Monitoring Rules

- Reassess security groups and IAM policies, enforce **least privilege**
- Improve detection rules for GuardDuty, Security Hub, and CloudWatch
- Use AWS Config and Service Control Policies (SCPs) to detect and prevent misconfigurations

