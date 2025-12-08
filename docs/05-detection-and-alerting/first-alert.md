# Detection Rules and Alerting

## Overview

This section documents the creation of automated detection rules and security dashboards for continuous threat monitoring.

## Alert Configuration

### Alert 1: Brute Force - Multiple Failed Login Attempts

**Purpose:** Detect potential password attack through multiple failed authentication attempts

**Search Query:**
```spl
index=windows sourcetype=WinEventLog:Security EventCode=4625 
| stats count by ComputerName, Account_Name 
| where count >= 3
```

**Configuration:**
- **Alert Type:** Scheduled
- **Schedule:** Every 5 minutes
- **Time Range:** Last 5 minutes
- **Trigger Condition:** Number of Results > 0
- **Trigger:** Once
- **Throttle:** 10 minutes
- **Actions:** Add to Triggered Alerts

**Detection Logic:**
1. Search for all failed logon events (EventCode 4625)
2. Count failures per computer and account
3. Alert if any account has 3+ failed attempts
4. Throttle prevents alert spam

**What This Detects:**
- Brute force password attacks
- Password spraying attempts
- Compromised credential attempts
- Legitimate users with forgotten passwords (requires context)

**Test Results:**
- ✅ Alert successfully triggered after 3 failed login attempts
- ✅ Displayed computer name and account name in alert
- ✅ Throttling prevented duplicate alerts

---

### Alert 2: New User Account Created

**Purpose:** Detect creation of new user accounts (potential backdoor activity)

**Search Query:**
```spl
index=windows sourcetype=WinEventLog:Security EventCode=4720
| table _time, ComputerName, Account_Name, user
```

**Configuration:**
- **Alert Type:** Real-time
- **Trigger Condition:** Number of Results > 0
- **Trigger:** For each result
- **Actions:** Add to Triggered Alerts

**Detection Logic:**
- Monitor for EventCode 4720 (user account created)
- Alert immediately when detected
- Capture account name and creator

**What This Detects:**
- Unauthorized account creation
- Backdoor persistence attempts
- Insider threat activity
- Administrative changes requiring review

**Test Results:**
- ✅ Alert triggered when test account created via `net user` command
- ✅ Real-time detection (< 2 minutes latency)

---

### Alert 3: User Added to Administrators Group

**Purpose:** Detect privilege escalation attempts

**Search Query:**
```spl
index=windows sourcetype=WinEventLog:Security EventCode=4732 Group_Name="Administrators"
| table _time, ComputerName, Account_Name, user
```

**Configuration:**
- **Alert Type:** Real-time
- **Trigger Condition:** Number of Results > 0
- **Trigger:** For each result
- **Actions:** Add to Triggered Alerts

**Detection Logic:**
- Monitor EventCode 4732 (member added to security group)
- Filter for "Administrators" group specifically
- Alert on any additions

**What This Detects:**
- Privilege escalation
- Unauthorized admin access
- Compromised accounts gaining elevated rights
- Insider threats

**Test Results:**
- ✅ Alert triggered when test user added to Administrators group
- ✅ Captured both the account added and who added it

---

## Security Dashboard

### Dashboard: Windows Security Monitoring

**Purpose:** Real-time visualization of security events and authentication activity

**Panels:**

#### Panel 1: Failed Login Attempts Timeline
**Search:**
```spl
index=windows sourcetype=WinEventLog:Security EventCode=4625 
| timechart count by ComputerName
```
**Visualization:** Line Chart  
**Time Range:** Last 24 hours

**Purpose:** 
- Identify spikes in failed authentications
- Track attack patterns over time
- Differentiate between isolated failures and sustained attacks

---

#### Panel 2: New Account Creations
**Search:**
```spl
index=windows sourcetype=WinEventLog:Security (EventCode=4720 OR EventCode=4726 OR EventCode=4732)
| eval Activity=case(
    EventCode=4720, "Account Created",
    EventCode=4726, "Account Deleted",
    EventCode=4732, "Added to Group"
)
| table _time, ComputerName, Activity, Account_Name
```
**Visualization:** Statistics Table  
**Time Range:** Last 7 days

**Purpose:**
- Track account lifecycle events
- Identify suspicious creation/deletion patterns
- Monitor privilege changes

---

#### Panel 3: Logon Types Distribution
**Search:**
```spl
index=windows sourcetype=WinEventLog:Security EventCode=4624 
| eval Logon_Description=case(
    Logon_Type=2, "Interactive (Console)",
    Logon_Type=3, "Network",
    Logon_Type=5, "Service",
    Logon_Type=7, "Unlock",
    Logon_Type=10, "Remote Desktop",
    1=1, "Other: ".Logon_Type
)
| stats count by Logon_Description
```
**Visualization:** Pie Chart  
**Time Range:** Last 24 hours

**Purpose:**
- Understand normal authentication patterns
- Detect anomalous logon methods (e.g., unexpected Remote Desktop)
- Baseline service vs. interactive logons

**Observations:**
- Majority of logons are Type 5 (Service) - expected for Windows background services
- Small percentage Type 2 (Interactive) - normal user console activity
- Absence of Type 10 (Remote Desktop) - appropriate for workstation without RDP enabled

---

## Windows Security Event Reference

### Critical Authentication Events

| EventCode | Event Name | Severity | Use Case |
|-----------|------------|----------|----------|
| **4624** | Successful logon | Info | Session tracking, baseline establishment |
| **4625** | Failed logon | High | Brute force detection, credential attacks |
| **4634** | Logoff | Info | Session duration analysis |
| **4647** | User-initiated logoff | Info | Normal logoff tracking |
| **4648** | Logon with explicit credentials | Medium | Lateral movement, pass-the-hash |
| **4672** | Special privileges assigned | Medium | Admin access monitoring |

### Account Management Events

| EventCode | Event Name | Severity | Use Case |
|-----------|------------|----------|----------|
| **4720** | User account created | High | Backdoor detection |
| **4722** | User account enabled | Medium | Dormant account activation |
| **4723** | Password change attempt | Medium | Credential manipulation |
| **4724** | Password reset attempt | Medium | Account takeover |
| **4725** | User account disabled | Low | Account lifecycle |
| **4726** | User account deleted | High | Evidence destruction |
| **4732** | Member added to security group | High | Privilege escalation |
| **4733** | Member removed from security group | Medium | Permission changes |

### Logon Type Reference

| Type | Description | Common Scenario | Suspicious Context |
|------|-------------|-----------------|-------------------|
| **2** | Interactive | User at keyboard | From service account |
| **3** | Network | File share access | Excessive failed attempts |
| **4** | Batch | Scheduled task | From user account |
| **5** | Service | Windows service | Normal (most common) |
| **7** | Unlock | Screen unlock | After-hours on locked workstation |
| **8** | NetworkCleartext | IIS authentication | Cleartext passwords |
| **10** | RemoteInteractive | RDP/Terminal Services | From unexpected IPs |
| **11** | CachedInteractive | Cached credentials | Offline authentication |

---

## Alert Tuning Considerations

### False Positive Scenarios

**Failed Login Alerts:**
- User legitimately forgot password (1-3 attempts = normal)
- Saved incorrect credentials in application
- Password expiration causing automated failures

**Mitigation:**
- Set threshold to 3+ failures (not 1-2)
- Add time window (5 minutes)
- Whitelist known service accounts

**New Account Creation:**
- Legitimate IT administration
- Automated provisioning systems
- Help desk operations

**Mitigation:**
- Alert on creation outside business hours
- Require correlation with ticketing system
- Monitor for creation + immediate privileged access

---

## Detection Effectiveness

### Metrics

**Coverage:**
- ✅ Brute force attacks (EventCode 4625)
- ✅ Account creation (EventCode 4720)
- ✅ Privilege escalation (EventCode 4732)
- ✅ Account deletion (EventCode 4726)

**Detection Latency:**
- Real-time alerts: < 2 minutes
- Scheduled alerts: 5-minute intervals
- Dashboard refresh: Real-time

**Test Results:**
- 3/3 alerts successfully triggered during testing
- 0 false positives observed
- 100% detection rate for simulated attacks

---

## Real-World Application

### SOC Analyst Workflow

**When Alert Fires:**

1. **Initial Triage (1-2 minutes)**
   - Review alert details (computer, account, count)
   - Check if source IP is internal or external
   - Verify user legitimacy

2. **Investigation (5-10 minutes)**
   - Search for related events from same source
   - Check for successful logon after failures
   - Review account history (recent changes, logon patterns)
   - Correlate with other security events

3. **Determination**
   - **False Positive:** Document reason, tune alert if needed
   - **True Positive:** Escalate to incident response
   - **Uncertain:** Continue investigation, gather more context

4. **Response Actions**
   - Lock account if compromised
   - Block source IP at firewall
   - Force password reset
   - Notify user/manager
   - Create incident ticket

---

## Key Takeaways

1. **Automated detection reduces response time** from hours/days to minutes
2. **Threshold-based alerting** (3+ failures) balances sensitivity and noise
3. **Real-time monitoring** enables immediate threat response
4. **Dashboards provide situational awareness** at a glance
5. **Context is critical** - same event can be normal or malicious based on who/when/where

---

## Future Enhancements

**Additional Detection Rules:**
- Multiple successful logons from different IPs (impossible travel)
- After-hours administrative activity
- Service account used for interactive logon
- Account locked out (EventCode 4740)
- Security log cleared (EventCode 1102)

**Dashboard Improvements:**
- Geographic map of logon sources
- Top users by failed attempts
- Account creation trend analysis
- Risk scoring based on multiple indicators

---

**Next:** [Advanced Threat Hunting](../06-threat-hunting/)
