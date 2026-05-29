## Analysis

### Why This Detection Works

Event ID 4625 is noisy (legitimate users generate dozens daily), 
but clustering is abnormal. The custom rule (Rule 100100) filters 
signal from noise:

**Threshold Logic:**
- 1-2 failures = user error (don't alert)
- 5 failures in 5 minutes = attack pattern (ALERT)
- 87 failures in 2 minutes = obvious attack (critical severity)

**Timeline of Detection:**
1. 10:32:00 - Hydra starts attack
2. 10:32:15 - 1st Event ID 4625 logged
3. 10:32:45 - 5th Event ID 4625 logged (Rule 100100 threshold met)
4. 10:32:46 - Wazuh generates alert
5. **45-second window for SOC analyst to respond** (disable account, block IP)
6. 10:34:00 - Hydra continues (but SOC can now take action)

**Without SIEM correlation:**
- Event Viewer shows 87 raw 4625 events
- Analyst manually reviews logs to spot pattern
- Takes 30+ minutes to notice attack
- Too late if attacker had valid credentials

**With SIEM correlation:**
- Wazuh automatically clusters failures
- Alert fires within 45 seconds
- SOC analyst sees Rule 100100 + source IP + workstation
- Can block or disable account while attack is ongoing

### Attack Indicators Observed

| Indicator | Value | Significance |
|-----------|-------|--------------|
| **Event ID** | 4625 | Failed authentication |
| **Count** | 87 (SMB) + 142 (RDP) | Coordinated attack, not user error |
| **Timeframe** | 2-5 minutes | Rapid fire = brute force, not typos |
| **Target** | edwin.lotsu | Specific account targeted (reconnaissance) |
| **Source IP** | 192.168.86.131 | Single attacker IP (or botnet during real attack) |
| **Logon Type** | 3 (Network) | Remote attack (RDP/SMB), not local |
| **Failure Code** | 0xC000006A | Correct username, wrong password = attacker knows user exists |
| **Workstation** | kali | Attack origin hostname visible in logs |

### Critical Insight: SubStatus Code 0xC000006A

This code means **"correct username, wrong password"** — significant because:
- Attacker already knows the username exists
- This is Stage 2 of attack (reconnaissance → password guessing)
- Not random probing (which would show code 0xC0000064 = user not found)
- Indicates attacker did reconnaissance first or using valid usernames

### False Positive Considerations

**When would this rule trigger legitimately?**
- User changes password, forgets new one → 3-4 attempts then remembers
- Service account password expired → automated retries
- VPN/RDP client with cached bad credentials → auto-retry attempts

**How to reduce false positives in production:**
- Whitelist known service accounts
- Adjust threshold to 10 failures (higher than lab's 5)
- Exclude known automated systems
- Add time-based filtering (alert only on business hours for office workers)

### What This Attack Revealed

**Detections working:** Event ID 4625 logged, custom rule fired, alert generated  
**Visibility gap fixed:** Without correlation rule, analyst would miss pattern  
**Not tested yet:** What if attacker tried multiple usernames? (password spraying)  
**Not tested yet:** What if attacker used distributed IPs? (botnet)

### Real SOC Response Workflow

If this alert fired in production:

1. **Immediate (1-2 min):** 
   - View alert: Rule 100100, target=edwin.lotsu, source=192.168.86.131
   - Query: Any successful logins from 192.168.86.131? (No = phew)
   - Decision: Disable account OR block IP

2. **Short-term (5-30 min):**
   - Force password reset for edwin.lotsu
   - Check if other accounts targeted from same IP
   - Query threat intel (AbuseIPDB, etc.)

3. **Long-term:**
   - Update firewall rules to block 192.168.86.131 RDP/SMB
   - Review if attacker accessed other systems
   - Adjust account lockout policy if too many legitimate failures
