# Wazuh SIEM Deployment - Troubleshooting Guide

This document covers real issues encountered during deployment and their solutions. These are **not theoretical** — these are actual problems I solved in my lab.

---

## Issue 1: Index Pattern Error in Dashboard

### Symptom
```
[Alerts index pattern] No template found for the selected index-pattern title [wazuh-alerts-*]
```

Dashboard shows "No results" even though agents are connected and `filebeat test output` passes.

### Root Cause
Wazuh index template not loaded into Elasticsearch, preventing proper indexing of alerts.

### Solution

**Step 1: Upload Wazuh template to Elasticsearch**
```bash
curl https://raw.githubusercontent.com/wazuh/wazuh/v4.4.5/extensions/elasticsearch/7.x/wazuh-template.json | \
curl -X PUT "https://192.168.86.130:9200/_template/wazuh" \
  -H 'Content-Type: application/json' \
  -d @- \
  -u logstash:YOUR_PASSWORD \
  -k
```

**Expected response:**
```json
{"acknowledged":true}
```

**Step 2: Verify template was created**
```bash
curl -X GET "https://192.168.86.130:9200/_template/wazuh" \
  -u logstash:YOUR_PASSWORD \
  -k
```

**Step 3: Restart Filebeat**
```bash
sudo systemctl restart filebeat
```

**Step 4: Check if index was created**
```bash
curl -X GET "https://192.168.86.130:9200/_cat/indices/wazuh-alerts-*?v" \
  -u logstash:YOUR_PASSWORD \
  -k
```

**Result:** Index appears and dashboard shows data.

---

## Issue 2: Filebeat SSL Certificate Verification Failure

### Symptom
```
x509: cannot validate certificate for 192.168.86.130 because it doesn't contain any IP SANs
```

Filebeat fails to connect to Wazuh Indexer due to SSL certificate mismatch.

### Root Cause
SSL certificate was generated for hostname (`siem-server`), but Filebeat config uses IP address (`192.168.86.130`).

### Solution (Option 1: Use Hostname)

**Step 1: Add hostname to `/etc/hosts`**
```bash
sudo nano /etc/hosts
```

Add:
```
192.168.86.130  siem-server
```

**Step 2: Update Filebeat config**
```bash
sudo nano /etc/filebeat/filebeat.yml
```

Change:
```yaml
hosts: ["192.168.86.130:9200"]
```

To:
```yaml
hosts: ["siem-server:9200"]
```

**Step 3: Test and restart**
```bash
sudo filebeat test output
sudo systemctl restart filebeat
```

### Solution (Option 2: Disable SSL Verification - Lab Only)

**Edit Filebeat config:**
```bash
sudo nano /etc/filebeat/filebeat.yml
```

Under the `ssl:` section, add:
```yaml
output.elasticsearch:
  hosts: ["192.168.86.130:9200"]
  protocol: https
  username: logstash
  password: YOUR_PASSWORD
  ssl:
    certificate_authorities:
      - /etc/filebeat/certs/root-ca.pem
    certificate: /etc/filebeat/certs/filebeat.pem
    key: /etc/filebeat/certs/filebeat-key.pem
    verification_mode: none  # ← ADD THIS LINE
```

**Test and restart:**
```bash
sudo filebeat test config
sudo filebeat test output
sudo systemctl restart filebeat
```

**Result:** Filebeat connects successfully, logs flow to dashboard.

---

## Issue 3: Wazuh Indexer Not Auto-Starting on Boot

### Symptom
After Ubuntu reboot, Wazuh dashboard shows "No data" until manually running:
```bash
sudo systemctl start wazuh-indexer
```

### Root Cause
Wazuh Indexer service not enabled for automatic startup.

### Solution

```bash
# Enable all Wazuh services
sudo systemctl enable wazuh-indexer
sudo systemctl enable filebeat
sudo systemctl enable wazuh-manager

# Verify they're enabled
sudo systemctl is-enabled wazuh-indexer
sudo systemctl is-enabled filebeat
sudo systemctl is-enabled wazuh-manager
```

**All should return:** `enabled`

**Test:** Reboot Ubuntu VM and verify services start automatically:
```bash
sudo reboot

# After reboot:
sudo systemctl status wazuh-indexer
sudo systemctl status filebeat
sudo systemctl status wazuh-manager
```

**Result:** All services start on boot without manual intervention.

---

## Issue 4: Sysmon Network Connection Logging Disabled

### Symptom
Port scan detection failing because Sysmon Event ID 3 (Network Connection) not being generated.

Running `sysmon64 -c` shows:
```
- Network connection:            disabled
```

### Root Cause
Sysmon config had empty `<RuleGroup>` with `onmatch="include"` but no rules defined:
```xml
<NetworkConnect onmatch="include">
    <!-- Empty - nothing to include -->
</NetworkConnect>
```

Sysmon interpreted this as "include nothing" → disabled the feature.

### Solution

**Edit Sysmon config:**
```powershell
notepad C:\Tools\Sysmon\sysmonconfig-export.xml
```

**Change from:**
```xml
<RuleGroup name="" groupRelation="or">
    <NetworkConnect onmatch="include">
        <!-- Empty -->
    </NetworkConnect>
</RuleGroup>
```

**To:**
```xml
<RuleGroup name="" groupRelation="or">
    <NetworkConnect onmatch="exclude">
        <!-- Log all network connections by excluding nothing -->
    </NetworkConnect>
</RuleGroup>
```

**Apply updated config:**
```powershell
cd C:\Tools\Sysmon
.\Sysmon64.exe -c sysmonconfig-export.xml
```

**Verify:**
```powershell
.\Sysmon64.exe -c
```

**Expected output:**
```
- Network connection:            enabled
```

**Test:**
```powershell
# Generate network activity
Test-NetConnection google.com -Port 443

# Check if logged
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5 | Where-Object {$_.Id -eq 3}
```

**Result:** Sysmon Event ID 3 now appears in logs, port scans detected in Wazuh.

---

## Issue 5: Agents Connected but No Logs in Dashboard

### Symptom
Wazuh Endpoints page shows "Active (2)" but Discover shows "No results found."

### Diagnostic Steps

**Step 1: Verify agents are actually active**
```bash
# On Wazuh Manager
sudo /var/ossec/bin/agent_control -l
```

**Expected:**
```
DESKTOP-7Y29ADQ is available.
WIN-HG03DC57KBN is available.
```

**Step 2: Check if agents are sending logs**
```bash
# On Wazuh Manager
sudo tail -f /var/ossec/logs/archives/archives.log
```

If you see logs scrolling → agents are sending data.

**Step 3: Check Filebeat status**
```bash
sudo systemctl status filebeat
```

If failed → check Issue #2 (SSL errors).

**Step 4: Check if Wazuh Indexer is running**
```bash
sudo systemctl status wazuh-indexer
```

If inactive → see Issue #3 (not auto-starting).

**Step 5: Check if indices exist**
```bash
curl -X GET "https://192.168.86.130:9200/_cat/indices?v" \
  -u logstash:YOUR_PASSWORD \
  -k
```

If no `wazuh-alerts-*` indices → see Issue #1 (index template).

---

## Quick Health Check Commands

### On Ubuntu SIEM Server

```bash
# Check all services
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status filebeat

# Check agent connections
sudo /var/ossec/bin/agent_control -l

# Check Filebeat connectivity
sudo filebeat test output

# Check if logs are flowing
sudo tail -f /var/ossec/logs/archives/archives.log

# Check indices
curl -X GET "https://192.168.86.130:9200/_cat/indices?v" -u logstash:PASSWORD -k
```

### On Windows Endpoints

```powershell
# Check Wazuh agent status
Get-Service WazuhSvc

# Check agent log for errors
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 20

# Restart agent if needed
Restart-Service WazuhSvc

# Check Sysmon status
Get-Service Sysmon64

# Verify Sysmon logging
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5
```

---

## Lessons Learned

1. **Always test Filebeat output** before assuming logs will flow
2. **Enable services for auto-start** immediately after installation
3. **Use hostnames instead of IPs** to avoid SSL certificate issues (or disable verification in lab)
4. **Empty Sysmon rules = disabled feature** — always define at least one rule
5. **Check each layer independently:** Agent → Manager → Filebeat → Indexer → Dashboard

---

## Useful Resources

- [Wazuh Troubleshooting Guide](https://documentation.wazuh.com/current/user-manual/manager/troubleshooting.html)
- [Filebeat Troubleshooting](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-installation-configuration.html)
- [Sysmon Configuration Guide](https://github.com/SwiftOnSecurity/sysmon-config)

---

[← Back to Deployment README](README.md)
