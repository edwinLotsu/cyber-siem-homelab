## Analysis

This simulation demonstrates how network reconnaissance activity can be detected using endpoint telemetry and SIEM correlation.

The Nmap TCP connect scan generated multiple rapid connection attempts to different service ports on the target system. These connections were logged by Sysmon (Event ID 3) and ingested into Wazuh, where a clear spike in activity was observed within a short time window.

The detection relies on identifying behavioral patterns rather than single events:
- One source IP
- Multiple destination ports
- High frequency within seconds

This approach aligns with MITRE ATT&CK T1046 (Network Service Scanning).

### Key Takeaways
- Endpoint telemetry (Sysmon) is essential for visibility into network activity
- SIEM correlation enables detection of patterns across multiple events
- Port scanning is noisy and easily detectable when proper logging is enabled
