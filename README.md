# splunk-homelab-soc-training
Building a SOC homelab with Splunk for log analysis and threat detection.


# Splunk Homelab SOC Training

## 🎯 Project Overview

This repository documents my journey from complete beginner to proficient Splunk analyst, building a functional Security Operations Center (SOC) in a homelab environment.

## 🔧 Lab Environment

**Virtualization Platform:** Oracle VirtualBox

**Virtual Machines:**
- **Splunk Server** - Ubuntu 24.04.3 LTS (6GB RAM, 50GB storage)
  - Static IP: 192.168.56.30
  - Splunk Enterprise 9.3.2
- **Windows 10 Workstation** (Planned)
- **Kali Linux** (Planned)

**Network Configuration:**
- Adapter 1: NAT (Internet access)
- Adapter 2: Host-only Adapter (Lab network)

## 📚 Learning Path

### ✅ Completed
- [x] Understanding what Splunk is and its use in cybersecurity
- [x] Installing Splunk Enterprise on Ubuntu
- [x] Configuring data receiving (port 9997)
- [x] Creating indexes for log storage
- [x] Understanding the Splunk interface
- [x] Learning to read and analyze logs
- [x] Basic search syntax and filtering
- [x] Installing Splunk Universal Forwarder on Windows
- [x] Configuring Windows Event Log collection
- [x] Understanding network adapter configurations (Host-only + NAT)
- [x] Analyzing Windows Security Events (EventCodes)
- [x] Generating and investigating test security events
- [x] Pattern recognition (failed logins, account creation/deletion)
- [x] Logon type analysis
- [x] Creating automated detection rules and alerts
- [x] Building security monitoring dashboards
- [x] Testing alert effectiveness with simulated attacks

### 🔄 In Progress
- [ ] Advanced SPL techniques (correlation, transactions)
- [ ] Attack simulation with Kali Linux
- [ ] Incident investigation workflows

## 📖 Documentation

Detailed guides for each phase:
- [01 - Introduction to Splunk](docs/01-introduction/)
- [02 - Installation & Setup](docs/02-installation/)
- [03 - Fundamentals](docs/03-fundamentals/)

## 🎓 Skills Developed

- Log analysis and interpretation
- Search Processing Language (SPL)
- Security event correlation
- Incident detection and response
- Dashboard creation and visualization

## 📝 Notes

This project is part of my cybersecurity learning journey. Each lesson builds on the previous one, starting from absolute beginner level.

---

**Last Updated:** November 15, 2025
