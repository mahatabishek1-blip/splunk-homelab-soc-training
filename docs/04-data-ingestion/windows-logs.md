# Windows Event Log Collection

## Overview

This guide documents the process of collecting Windows Event Logs using Splunk Universal Forwarder and analyzing security events.

## Lab Environment

**Windows 10 VM:**
- Resources: 6GB RAM, 50GB Storage
- Network: Host-only Adapter (Primary) + NAT (Secondary)
- Static IP: 192.168.56.10

**Splunk Server:**
- Ubuntu 24.04.3 LTS
- IP: 192.168.56.30
- Receiving Port: 9997

## Network Configuration Explained

### Why Two Network Adapters?

**Adapter 1: Host-Only Adapter (192.168.56.x)**
- Creates isolated private network between VMs and host
- Enables secure lab communication
- Splunk log forwarding uses this network
- Prevents lab activity from affecting home/production networks

**Adapter 2: NAT**
- Provides internet access for downloads and updates
- VM appears as client behind host's IP
- Blocks unsolicited incoming connections (security)

**Benefits:**
- Isolated lab environment (safe for malware/attack testing)
- Internet access when needed
- Controlled traffic flow

## Universal Forwarder Installation

### Step 1: Download

1. Visit: https://www.splunk.com/en_us/download/universal-forwarder.html
2. Download Windows 64-bit version
3. File: `splunkforwarder-9.3.2-xxx-x64-release.msi`

### Step 2: Install

**Installation Settings:**
- Username: `admin`
- Password: (secure password)
- Deployment Server: (leave blank)
- **Receiving Indexer:** `192.168.56.30`
- **Port:** `9997`
- Install Location: `C:\Program Files\SplunkUniversalForwarder`

### Step 3: Configure Firewall

**Create Outbound Rule:**
1. Open: `wf.msc` (Windows Defender Firewall with Advanced Security)
2. Outbound Rules → New Rule
3. Type: Port (TCP), Remote Port: 9997
4. Action: Allow
5. Profile: All (Domain, Private, Public)
6. Name: "Splunk Universal Forwarder"

### Step 4: Configure Data Inputs

**Method: Configuration File (Recommended)**

Navigate to config directory:
```cmd
cd "C:\Program Files\SplunkUniversalForwarder\etc\system\local"
```

Create/edit `inputs.conf`:
```cmd
notepad inputs.conf
```

**Configuration:**
```ini
[tcpout]
defaultGroup = default-autolb-group

[tcpout:default-autolb-group]
server = 192.168.56.30:9997

[WinEventLog://Security]
disabled = false
index = windows
sourcetype = WinEventLog:Security
renderXml = true

[WinEventLog://System]
disabled = false
index = windows
sourcetype = WinEventLog:System
renderXml = true

[WinEventLog://Application]
disabled = false
index = windows
sourcetype = WinEventLog:Application
renderXml = true
```

Save and restart the service:
```cmd
net stop SplunkForwarder
net start SplunkForwarder
```

## Verification

### Check Forwarder Status
```cmd
cd "C:\Program Files\SplunkUniversalForwarder\bin"
splunk list inputstatus -auth admin:PASSWORD
```

**Look for:**
```
C:\Program Files\SplunkUniversalForwarder\bin\splunk-winevtlog.exe
time opened = [timestamp]
total bytes = [number]
```

### Verify in Splunk

Search in Splunk web interface:
```
index=windows
```

Check sourcetypes:
```
index=windows | stats count by sourcetype
```

**Expected sourcetypes:**
- `WinEventLog:Security`
- `WinEventLog:System`
- `WinEventLog:Application`

## Windows Security Event Analysis

### Test Event Generation

#### Failed Login Attempts
1. Lock screen (Windows + L)
2. Enter wrong password 5 times
3. Log in correctly

**Search:**
```
index=windows sourcetype=WinEventLog:Security EventCode=4625
| table _time, ComputerName, Account_Name, Failure_Reason
```

**Observed Results:**
- 5 failed login events detected
- Failure Reason: "Unknown user name or bad password"
- EventCode 4625 (Failed logon)

#### Account Creation/Deletion
```cmd
net user testuser Password123! /add
net user testuser /delete
```

**Search:**
```
index=windows sourcetype=WinEventLog:Security (EventCode=4720 OR EventCode=4726)
| table _time, EventCode, Account_Name, user
```

**Observed Results:**
- EventCode 4720: Account created at 21:26:00
- EventCode 4726: Account deleted at 21:26:15
- Time gap: 15 seconds (simulates attacker creating backdoor and covering tracks)

#### Successful Logon Analysis
```
index=windows sourcetype=WinEventLog:Security EventCode=4624
| table _time, Account_Name, Logon_Type, Workstation_Name
```

**Observed Results:**
- 72 successful logon events
- Majority: Logon Type 5 (Service) - Background Windows services
- Few: Logon Type 2 (Interactive) - Console logons

## Critical Windows Event Codes

### Authentication Events

| EventCode | Event Name | Significance |
|-----------|------------|--------------|
| **4624** | Successful logon | Track user access |
| **4625** | Failed logon | Brute force detection |
| **4634** | Logoff | Session tracking |
| **4648** | Explicit credential logon | Lateral movement |
| **4672** | Special privileges assigned | Admin access monitoring |

### Account Management

| EventCode | Event Name | Significance |
|-----------|------------|--------------|
| **4720** | User account created | Backdoor detection |
| **4722** | User account enabled | Dormant account activation |
| **4724** | Password reset attempt | Credential attacks |
| **4726** | User account deleted | Covering tracks |
| **4732** | User added to security group | Privilege escalation |

### Logon Types

| Type | Description | Common Scenario |
|------|-------------|-----------------|
| **2** | Interactive | Console login |
| **3** | Network | File share access |
| **5** | Service | Background services |
| **7** | Unlock | Screen unlock |
| **10** | RemoteInteractive | Remote Desktop |

## Analysis Examples

### Example 1: Privilege Assignment (EventCode 4672)

**Event Details:**
- EventCode: 4672
- Account: SYSTEM
- Time: 21:34:00
- Interpretation: Windows SYSTEM account granted administrative privileges

**Analysis:** Normal behavior. SYSTEM account requires elevated privileges for core Windows services.

**Red Flag:** If this event shows a regular user account (e.g., "john.smith"), investigate why they have admin privileges.

### Example 2: Failed Login Pattern (EventCode 4625)

**Event Details:**
- EventCode: 4625
- Count: 5 attempts
- Pattern: Rapid succession
- Failure Reason: Unknown user name or bad password

**Analysis:** Multiple failed attempts indicate possible brute force attack. In production, 3-5 failures typically trigger automated alerts.

### Example 3: Account Lifecycle (EventCode 4720 → 4726)

**Timeline:**
- 21:26:00 - EventCode 4720 (Account "hacker" created)
- 21:26:15 - EventCode 4726 (Account "hacker" deleted)

**Analysis:** 15-second gap between creation and deletion is suspicious. This pattern suggests:
- Attacker created backdoor account
- Used it briefly for persistence
- Deleted to cover tracks

**Investigation steps:**
- Check for logon events (4624) for this account
- Review file access/modifications during 15-second window
- Check for privilege escalation (4732)

## Key Takeaways

1. **Pattern Recognition:** Normal vs suspicious activity requires understanding baselines
2. **Context Matters:** Same event can be normal or critical depending on who/when/where
3. **Timeline Analysis:** Correlating events reveals attack chains
4. **Privilege Monitoring:** Track administrative access and account changes
5. **Logon Type Analysis:** Different logon types indicate different access methods

## Troubleshooting

### No Data Appearing in Splunk

**Check forward-server configuration:**
```cmd
splunk list forward-server -auth admin:PASSWORD
```

Should show: `192.168.56.30:9997`

**If missing, add:**
```cmd
splunk add forward-server 192.168.56.30:9997 -auth admin:PASSWORD
net stop SplunkForwarder
net start SplunkForwarder
```

**Check Ubuntu firewall:**
```bash
sudo ufw allow 9997/tcp
sudo ufw reload
```

**Review forwarder logs:**
```cmd
notepad "C:\Program Files\SplunkUniversalForwarder\var\log\splunk\splunkd.log"
```

Look for connection errors or authentication failures.

---

**Next:** [Detection Rules and Alerting](../05-detection-and-alerting/first-alert.md)
