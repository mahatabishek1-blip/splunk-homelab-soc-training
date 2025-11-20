# Splunk Enterprise Installation Guide

## Environment Details

**Virtual Machine:** Ubuntu 24.04.3 LTS  
**Resources:** 6GB RAM, 50GB Storage  
**Network Configuration:**
- Adapter 1: NAT (Internet access)
- Adapter 2: Host-only Adapter (Lab network - 192.168.56.0/24)
- Static IP: 192.168.56.30

## Installation Steps

### 1. System Preparation

Update the system:
```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Download Splunk Enterprise

Create installation directory:
```bash
cd ~
mkdir splunk-install
cd splunk-install
```

Download Splunk Enterprise 9.3.2:
```bash
wget -O splunk-9.3.2-d8bb32809498-linux-2.6-amd64.deb "https://download.splunk.com/products/splunk/releases/9.3.2/linux/splunk-9.3.2-d8bb32809498-linux-2.6-amd64.deb"
```

### 3. Install Splunk
```bash
sudo dpkg -i splunk-9.3.2-d8bb32809498-linux-2.6-amd64.deb
```

### 4. Initial Configuration

Start Splunk and accept license:
```bash
cd /opt/splunk/bin
sudo ./splunk start --accept-license
```

**Important:** Create administrator credentials during first startup.

### 5. Enable Auto-Start on Boot
```bash
sudo /opt/splunk/bin/splunk enable boot-start
```

### 6. Configure Data Receiving

Enable port 9997 to receive data from forwarders:
```bash
sudo /opt/splunk/bin/splunk enable listen 9997 -auth admin:YOUR_PASSWORD
```

### 7. Create Windows Index
```bash
sudo /opt/splunk/bin/splunk add index windows -auth admin:YOUR_PASSWORD
```

### 8. Configure Firewall (if enabled)
```bash
sudo ufw allow 8000/tcp  # Web interface
sudo ufw allow 9997/tcp  # Data receiving
sudo ufw reload
```

## Accessing Splunk

**Web Interface:** http://192.168.56.30:8000

**Default Ports:**
- 8000: Web interface
- 9997: Data receiving (forwarders)
- 8089: Management port

## Verification

Check Splunk status:
```bash
sudo /opt/splunk/bin/splunk status
```

Verify receiving port:
```bash
sudo /opt/splunk/bin/splunk list inputstatus -auth admin:YOUR_PASSWORD
```

## Key Locations

- **Installation Directory:** `/opt/splunk/`
- **Configuration Files:** `/opt/splunk/etc/`
- **Logs:** `/opt/splunk/var/log/splunk/`
- **Indexes:** `/opt/splunk/var/lib/splunk/`

---

**Next:** [Windows Log Collection](../04-data-ingestion/windows-logs.md)
