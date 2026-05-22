# Attack Simulation: Brute Force Detection (SMB + RDP)
## Scenario Overview

**Objective:** Detect brute force authentication attempts using Windows Event Logs and SIEM correlation

**Attacker:** Kali Linux
**Target:** Windows 10 Client  
**Attack Tool:** Hydra (SMB and RDP modules)  
**Detection Tools:** Wazuh SIEM, Custom Rule ID 100100, Windows Security Event Log (Event ID 4625)

**MITRE ATT&CK Technique:** [T1110.001 - Brute Force: Password Guessing](https://attack.mitre.org/techniques/T1110/001/)

---

## Attack Execution

## SMB Brute Force

### Attack Command
```bash
hydra -l edwin.lotsu -P ~/small-wordlist.txt 192.168.86.129 smb2
```

**Execution Method:**
- Hydra attempted SMB authentication using a predefined password list
- Multiple login attempts targeted the Windows client over TCP/445
- Simulated password guessing against SMB authentication

**Result**
- No valid password found
- Multiple failed authentication attempts generated

 **Screenshot:**

 ---

## RDP Brute Force

## Service Validation

### Port Scan
```bash
nmap -p 3389 192.168.86.129   #Confirming Remote Desktop service availability before attack execution
```

### Attack Command
```bash
hydra -l edwin.lotsu -P ~/small-wordlist.txt -t 1 -W 3 192.168.86.129 rdp
```

**Execution Method:**
- Hydra initiated repeated RDP authentication attempts
- Password list used against target user account
- Simulated remote credential guessing attack

**Result**
- No valid credentials identified
- Authentication failures recorded on Windows target

 **Screenshot:**

---

## Detection Strategy

### Data Sources
1. Window Security Event Log
2. Wazuh SIEM
3. Custom Wazuh Correlation

### Windows Event Detection
Primary Detection:
```
Event ID 4625
```

Meaning: An account failed to log on

### Wazuh Dicover Queries
**Raw Failed Logons**
```
data.win.system.eventID:4625
```

**Targeted Account**
```
data.win.eventdata.targetUserName: edwin.lotsu
```
**Attacker IP**
```
data.win.eventdata.ipAddress: 192.168.86.131
```
**Custom rule correlation**
```
rule.id: 100100
```

## Custom Rule Correlation
**Custom rule configured in:**
```
local_rules.xml
```

**Rule:**
```xml
<rule id="100100" level="10" frequency="5" timeframe="300">
  <if_matched_sid>60122</if_matched_sid>
  <same_field>win.eventdata.targetUserName</same_field>
  <description>
    Multiple failed Windows logon attempts for same user
  </description>
</rule>
```

### Detection Logic
- Trigger afer 5 failed logons
- 5-minute timeframe
- Same target account correlation
- Elevates repeated failures beyond raw Windows logging

**Screenshot:**

---

## Detection Results
### Windows Event Viewer
Event Viewer recorded repeated:
```
Event ID 4625
```
Observed fields:
| Field | Value |
|-------|-------|
|**Event ID** | 4625 |
|**Target User** | edwin.lotsu |
|**Logon Type** | 3 |
|**Authentication** | NTLM |
|**Workstation** | kali |
|**Source IP** | 192.168.86.131 |

**Meaning**
- Authentication attempts originated from a Kali Linux endpoint
- Targeted Remote authentication
- Failed Repeatedly

**Screenshot**
---

### Wazuh SIEM Correlation
Wazuh successsfully ingested and correlated the attack.
Observed:
| Field | Value |
|-------|-------|
|**Rule ID** | 100100 |
|**Description** | Multiple failed Windows logon attempts for same user |
|**Event ID** | 4625 |
|**Target User** | edwin.lotsu |
|**Source IP** | 192.168.86.131 |
|**Workstation** | kali |
|**Logon Type** | 3 |

The attack was elevated from *single login failures* to **correlated brute force activity** 

**Screenshots**

## Validation Tests

### Test 1 – SMB Authentication Failures

Hydra SMB attack generated:

- Failed authentication attempts
- Event ID 4625
- Wazuh ingestion verified

**Result:**  
Successful detection

---

### Test 2 – RDP Authentication Failures

Hydra RDP attack generated:

- Remote authentication failures
- Attacker workstation visibility
- Source IP visibility
- Wazuh custom rule correlation

**Result:**  
Successful detection

---

### Test 3 – Rule Correlation

**Query:**
```text
rule.id:100100
```

**Result:**

- Multiple correlated alerts
- Same target account
- Threshold exceeded

---

## Analysis

### Why This Detection Works

- Windows natively logs failed authentication
- Wazuh centralizes visibility
- Custom rules correlate behavior instead of isolated events
- Source IP and workstation context identify attacker origin

---

### Attack Indicators Observed

| Indicator | Value | Significance |
|---|---|---|
| Event ID | 4625 | Failed authentication |
| Rule ID | 100100 | Correlated brute force |
| Logon Type | 3 | Remote/network logon |
| Source IP | 192.168.86.131 | Attacker host |
| Workstation | kali | Attack source |
| Failure Count | Multiple | Password guessing behavior |

---

## Recommended Response Actions

### SOC Workflow

1. Verify legitimacy of login attempts  
2. Identify source IP and workstation  
3. Review successful logons after failures  
4. Assess privilege level of targeted account  
5. Block or isolate source if attack persists  

---

### Recommended Hardening

- Account lockout policy
- MFA
- RDP restrictions
- Wazuh alert automation
- Network segmentation

---

## Lessons Learned

1. Event ID 4625 alone is noisy  
2. Correlation rules improve visibility  
3. Source IP and workstation data are critical  
4. SIEM provides context that Event Viewer alone cannot  

---

## Related Scenarios

- [01 – Nmap Port Scan Detection](../01-nmap-port-scan/)
- [02 – Failed Login Detection](../02-failed-login-detection/)
