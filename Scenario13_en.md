<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [IAM MFA and Session Duration Confusion Points Summary](#iam-mfa-and-session-duration-confusion-points-summary)
  - [🔄 Common Confusions: `aws:MultiFactorAuthAge` vs. `MaxSessionDuration`](#-common-confusions-awsmultifactorauthage-vs-maxsessionduration)
    - [🔐 `aws:MultiFactorAuthAge`](#-awsmultifactorauthage)
    - [🕒 `MaxSessionDuration`](#-maxsessionduration)
    - [🧩 Common Misconceptions and Clarifications](#-common-misconceptions-and-clarifications)
  - [🔎 Other Related Time Keys (for reference)](#-other-related-time-keys-for-reference)
  - [✅ Conclusion: Use Time Keys Appropriately Based on Context](#-conclusion-use-time-keys-appropriately-based-on-context)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


#  IAM MFA and Session Duration Confusion Points Summary

---

## 🔄 Common Confusions: `aws:MultiFactorAuthAge` vs. `MaxSessionDuration`

When designing IAM policies involving MFA and session control, several similar-sounding parameters can easily be confused. Here's a clear breakdown:

### 🔐 `aws:MultiFactorAuthAge`

| Item | Description |
|------|-------------|
| Type | IAM policy `Condition` key |
| Unit | Seconds (e.g., 7200 seconds = 2 hours) |
| Meaning | Represents how much time has passed since the user authenticated with MFA |
| Usage | `NumericLessThan: { "aws:MultiFactorAuthAge": 7200 }` enforces actions only if the session is less than 2 hours old |
| Applies to | **IAM users authenticated with MFA** |

### 🕒 `MaxSessionDuration`

| Item | Description |
|------|-------------|
| Type | IAM Role configuration setting (not used in `Condition`) |
| Unit | Seconds (settable between 3600–43200) |
| Meaning | Defines how long a role session (assumed via `AssumeRole`) can last |
| Usage | Set directly in the role configuration, not in policy conditions |
| Applies to | **IAM roles** assumed via `AssumeRole` API call |

---

### 🧩 Common Misconceptions and Clarifications

| Confused Item | Clarification |
|---------------|---------------|
| Writing `MaxSessionDuration` in policy `Condition` | ❌ Not valid. This is a role-level setting, not a policy condition. |
| Applying `aws:MultiFactorAuthAge` to role sessions | ⚠️ Generally applies to IAM users only. MFA context might not carry over to assumed roles. |
| Using `aws:TokenIssueTime` instead | ✅ Can be a workaround to evaluate session age, but `MultiFactorAuthAge` is more direct and readable. |

---

## 🔎 Other Related Time Keys (for reference)

| Key | Meaning | Notes |
|-----|---------|-------|
| `aws:TokenIssueTime` | Time when the session was issued | Alternative to MFA age checks if unavailable |
| `aws:CurrentTime` | Current time (UTC) | Useful for time-bound access control |
| `aws:EpochTime` | Current UNIX timestamp | Enables arithmetic or range calculations on time values |

---

## ✅ Conclusion: Use Time Keys Appropriately Based on Context

- ✅ **For MFA-based limits (e.g., "within 2 hours of login") → use `aws:MultiFactorAuthAge`**
- ✅ **For role session duration → set `MaxSessionDuration` on the role**
- ✅ Only certain time-based keys can be used in IAM policy `Condition` blocks, such as `MultiFactorAuthAge`, `CurrentTime`, etc.
