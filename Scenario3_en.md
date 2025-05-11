
# AWS EC2 × SSM: Secure Incident‑Response Workflow (Memory Preservation & Isolation)

---

## 📘 Scenario

A company suspects that an Amazon EC2 instance running **AWS Systems Manager Agent (SSM Agent)** has been compromised.  
All instances are EBS‑backed. During the investigation the instance **must stay online** while the workflow satisfies these requirements:

1. 🔒 **Preserve volatile memory (RAM) and non‑volatile memory (EBS) data.**
2. 🏷️ **Tag the instance with the incident‑ticket information.**
3. 🔗 **Isolate the instance to stop malware spreading yet keep it reachable for analysts.**
4. 📝 **Record every investigative action (especially RAM collection) for forensics.**

---

## 🔎 Investigation Workflow

1. **Detect** – GuardDuty, EDR, CloudTrail anomalies.  
2. **Initial triage** – collect instance ID, VPC, severity, open ticket.  
3. **Isolation & preservation** – change SG, detach from ASG/ELB, enable Termination Protection.  
4. **Collect volatile data** – run SSM Run Command.  
5. **Disk preservation** – create EBS snapshot, add ticket tags.  
6. **Deep analysis** – copy snapshot to isolated account, mount read‑only.  
7. **Root‑cause & containment** – identify malware, stolen creds, patch.  
8. **Recovery & lessons learned** – rebuild AMI, improve detection rules.

---

## 🛡️ Isolation Strategy

| Item to isolate | Purpose | Action |
|-----------------|---------|--------|
| **Network traffic** | Stop C2 / lateral movement | Attach a quarantine security group that denies all inbound / restricts outbound. |
| **Auto Scaling Group** | Avoid respawning infected AMIs | `detach-instances --should-decrement-desired-capacity`. |
| **Elastic Load Balancer** | Block malicious traffic to users | `deregister-instances-from-load-balancer`. |
| **Lifecycle ops** | Prevent evidence loss | **Enable Termination Protection** (`modify-instance-attribute --disable-api-termination false`). |

**Termination Protection** prevents accidental Stop/Terminate; disabling it later leaves an audit trail.

---

## 🔧 Security‑Group Hardening

| Port | Before | After isolation |
|------|--------|-----------------|
| 22 / 3389 | Office IPs allowed | **Deny all** |
| 80 / 443 | Public | **Deny all** |
| Others | App dependent | **Deny all** |

SSM Agent needs only **outbound 443 HTTPS**, so inbound can be zero.

---

## 🚀 Collecting Volatile Data with **SSM Run Command**

```bash
aws ssm send-command   --instance-ids i-xxxxxxxx   --document-name "AWS-RunShellScript"   --comment "Forensic RAM dump INC-123456"   --parameters 'commands=["date -u","cat /proc/meminfo","netstat -pantue","ps aux --sort=-%mem | head -n 30"]'   --output-s3-bucket-name forensic-logs-bucket   --output-s3-key-prefix "INC-123456/volatile"
```

### Why **Run Command** over SSH/RDP?

| Aspect | Run Command | SSH / RDP session |
|--------|------------|-------------------|
| Inbound ports | **None** | Need 22/3389 open |
| Audit trail | CloudTrail & SSM logs | OS logs only, incomplete |
| Automation | Parameterized, repeatable | Manual steps, typos risk |
| Credential mgmt | IAM role only | Keys/passwords to manage |
| Ops overhead | Very low | High (bastion, firewall rules) |

---

## 💾 Creating an EBS Snapshot & Tagging

```bash
SNAP_ID=$(aws ec2 create-snapshot   --volume-id vol-xxxxxxxx   --description "INC-123456 evidence"   --query SnapshotId --output text)

aws ec2 create-tags   --resources "$SNAP_ID" i-xxxxxxxx vol-xxxxxxxx   --tags Key=IncidentTicket,Value=INC-123456 Key=Evidence,Value=Yes
```

Copy the snapshot to another account/region to mitigate deletion risk.

---

## ❌ Why Options **B / D / F** Are Not Optimal

### Option B – move instance to an isolation subnet
* Changes private IP / EIP, route tables; high blast radius.  
* SG update (Option A) isolates without network re‑plumbing → lower overhead.

### Option D – open SSH/RDP and run scripts
* Requires opening ports, managing creds, session recording.  
* Run Command already available via SSM Agent → safer & cheaper.

### Option F – State Manager association to create snapshot
* State Manager is for **continuous configuration enforcement**.  
* One‑off snapshot is faster with a direct API call (Option E); association creation/cleanup adds complexity.

---

## 📚 Extras

### What is an **RDP session**?
Microsoft Remote Desktop Protocol lets admins connect interactively to Windows servers for GUI‑based tasks (event viewer, registry editing, etc.). Requires port 3389 and is heavy for incident forensics.

### Other common **Run Command** use cases
* Ad‑hoc patch validation on a subset of servers  
* Bulk log extraction to S3  
* Rolling restarts of web services across regions  
* Inventory collection of installed packages

### Typical **State Manager** scenarios
| Scenario | Example association |
|----------|--------------------|
| Patch compliance | `AWS-ApplyPatchBaseline` nightly at 02:00 JST |
| Drift remediation | Re‑install missing security agent |
| CIS hardening | Enforce registry or sysctl settings |

An **Association** ties a document, target instances, schedule and parameters – ideal for continuous, not one‑time, tasks.

---

## 🔐 Security Checklist

- Termination Protection enabled  
- Quarantine SG denies all inbound  
- Run Command executions logged in CloudTrail  
- Snapshot encrypted & tagged  
- Evidence in S3 with versioning / object‑lock

---

With this workflow the security team can **preserve evidence, isolate the host and investigate** without shutting down the instance, while keeping operational overhead minimal and maintaining a verifiable audit trail. 💡
