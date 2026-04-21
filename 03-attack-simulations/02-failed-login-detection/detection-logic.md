## Detection Logic

Brute force activity was identified using repeated failed authentication attempts.

### Detection Query
data.win.system.eventID: 4625 AND data.win.eventdata.targetUserName: Administrator AND agent.ip: 192.168.86.129

### Key Indicators

- Multiple Event ID 4625 logs
- Same target account (Administrator)
- Same source machine
- High frequency within short time window

### Detection Approach

1. Monitor Windows Security logs for Event ID 4625
2. Filter by high-value accounts (e.g., Administrator)
3. Identify repeated failures within a short time period
4. Correlate failures without successful login (Event ID 4624)

### Threshold Example

- 5+ failed logins within 5 minutes → trigger alert

### MITRE Mapping

- T1110.001 — Brute Force: Password Guessing

### Detection Limitations

- Slow brute force attacks may evade detection
- Password spraying spreads attempts across accounts
- Legitimate user mistakes may generate noise
